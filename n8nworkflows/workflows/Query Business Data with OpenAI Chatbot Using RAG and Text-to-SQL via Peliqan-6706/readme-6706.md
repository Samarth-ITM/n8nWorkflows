Query Business Data with OpenAI Chatbot Using RAG and Text-to-SQL via Peliqan

https://n8nworkflows.xyz/workflows/query-business-data-with-openai-chatbot-using-rag-and-text-to-sql-via-peliqan-6706


# Query Business Data with OpenAI Chatbot Using RAG and Text-to-SQL via Peliqan

---

### 1. Workflow Overview

This workflow implements an AI-powered chatbot designed to query internal business data using Retrieval-Augmented Generation (RAG) combined with Text-to-SQL capabilities, facilitated via Peliqan's data platform. It serves use cases where users want to ask natural language questions about business data spanning multiple sources (e.g., Notion, Chargebee, Hubspot) and receive accurate, data-driven answers by leveraging both vector search and direct SQL query execution.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures chat messages from users to trigger the workflow.
- **1.2 AI Agent Processing:** Uses a LangChain AI Agent configured with system instructions and tools to interpret user queries, decide between retrieval and SQL execution, and generate responses.
- **1.3 Tools for AI Agent:** Defines external tools available to the Agent, including vector store retrieval for RAG and Peliqan SQL execution.
- **1.4 Data Feeding for RAG:** Processes and stores business data into a vector store (Supabase) to enable efficient semantic search.
- **1.5 Supporting Services:** Includes memory management for chat context and OpenAI model configurations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming chat messages from users via a webhook, triggering the AI chatbot workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger Node  
    - Role: Entry point for chat messages; initiates the workflow when a user sends a message.  
    - Configuration: Public webhook enabled with default options, allowing external calls.  
    - Inputs: External HTTP webhook call with chat message payload.  
    - Outputs: Passes chat message data to the AI Agent node.  
    - Edge cases: Potential webhook timeout or malformed message payloads.  
    - Version: 1.1  

#### 2.2 AI Agent Processing

- **Overview:**  
  Core logic that manages user query interpretation, decides whether to perform vector search or execute SQL, and generates the chatbot's response. It incorporates system instructions about data sources and tools, maintains chat context, and outputs final answers.

- **Nodes Involved:**  
  - AI Agent  
  - Simple Memory  
  - OpenAI Chat Model

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Orchestrates processing of user input using defined tools and memory, guided by a detailed system prompt.  
    - Configuration:  
      - System Message includes instructions that the assistant can:  
        - Use vector store for info retrieval from Notion pages (internal knowledge).  
        - Run SQL queries via Peliqan tool "Execute SQL" on specified data tables with known schema.  
      - Always outputs data regardless of internal node state.  
    - Inputs: Chat message from "When chat message received", memory buffer, and AI language model.  
    - Outputs: Final chatbot response and tool calls (SQL queries, vector store requests).  
    - Edge cases: Misinterpretation of SQL syntax, tool call failures, or missing data in vector store.  
    - Version: 1.6  

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains chat history context for the AI Agent to provide coherent multi-turn conversations.  
    - Configuration: Default (no window size specified, but typically recent messages buffered).  
    - Inputs: Connected to AI Agent's memory input.  
    - Outputs: Provides chat history data to AI Agent.  
    - Edge cases: Memory overflow or loss of relevant context for long conversations.  
    - Version: 1.3  

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides the underlying language model for text generation in the AI Agent.  
    - Configuration: Default OpenAI model settings; uses configured OpenAI API credentials.  
    - Inputs: Receives prompts from AI Agent.  
    - Outputs: Generated chat completions passed back to AI Agent.  
    - Credentials: OpenAI API key required (configured).  
    - Edge cases: API rate limits, network errors, or prompt formatting issues.  
    - Version: 1  

#### 2.3 Tools for AI Agent: SQL and RAG

- **Overview:**  
  Defines the external data access tools the AI Agent can invoke: a Peliqan SQL executor and a Supabase vector store for RAG retrieval.

- **Nodes Involved:**  
  - Execute an SQL query via Peliqan  
  - Supabase Vector Store for search  
  - Embeddings OpenAI for RAG retrieval  
  - Sticky Note (Tools for AI Agent: SQL & RAG)

- **Node Details:**

  - **Execute an SQL query via Peliqan**  
    - Type: Peliqan Tool Node  
    - Role: Executes SQL queries generated by the AI Agent on the Peliqan data warehouse.  
    - Configuration:  
      - Operation: Execute SQL query on Peliqan resource "query".  
      - Query SQL: Uses dynamic input from AI Agent's SQL_Query variable.  
      - Connection: Uses configured Peliqan credentials.  
    - Inputs: SQL query string from AI Agent tool call.  
    - Outputs: SQL query results back to AI Agent.  
    - Credentials: Peliqan API key required.  
    - Edge cases: SQL syntax errors, Peliqan API errors, connection issues.  
    - Version: 1  
    - Note: Requires environment variable `N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=True` to enable tool usage.  

  - **Supabase Vector Store for search**  
    - Type: LangChain Vector Store Supabase (retrieve-as-tool mode)  
    - Role: Provides semantic search capability over internal knowledge base documents (e.g., Notion content) for the AI Agent.  
    - Configuration:  
      - Mode: Retrieve as tool with a friendly name "Company_knowledge_base".  
      - Table: Uses the "documents" table in Supabase.  
      - Tool Description: Used to find internal knowledge from Notion or similar sources.  
    - Inputs: AI Agent tool calls for retrieval.  
    - Outputs: Search results to AI Agent.  
    - Credentials: Supabase API credentials required.  
    - Edge cases: Vector store query failures, missing or outdated data.  
    - Version: 1.1  

  - **Embeddings OpenAI for RAG retrieval**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates embeddings for vector store search queries.  
    - Configuration: Default OpenAI embeddings settings.  
    - Inputs: Connected to Supabase Vector Store for search node for embedding queries.  
    - Outputs: Embeddings passed to vector store.  
    - Credentials: OpenAI API key required.  
    - Edge cases: API errors, rate limits.  
    - Version: 1  

#### 2.4 Data Feeding for RAG

- **Overview:**  
  Loads and processes business data documents, splits them into text chunks, generates embeddings, and inserts them into the Supabase vector store, enabling the RAG retrieval functionality.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get table data (Peliqan)  
  - Default Data Loader  
  - Recursive Character Text Splitter  
  - Embeddings OpenAI for RAG  
  - Supabase Vector Store to store vectors for RAG  
  - Sticky Note ("Feed data into Vector Store (for RAG)")

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution to feed data into vector store.  
    - Inputs: Manual user trigger.  
    - Outputs: Starts "Get table data" node.  
    - Edge cases: None.

  - **Get table data**  
    - Type: Peliqan Node  
    - Role: Retrieves business data from Peliqan's data warehouse for RAG ingestion.  
    - Configuration:  
      - Table ID: 200653 (specific data table to load).  
    - Inputs: Triggered by manual node.  
    - Outputs: Passes raw data to Default Data Loader.  
    - Credentials: Peliqan API key required.  
    - Edge cases: API errors, empty data sets.  
    - Version: 1

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Converts raw JSON data into documents suitable for embedding.  
    - Configuration:  
      - Metadata fields: Uses `_source_url` and `title` from JSON.  
      - Document content formed by combining title and content fields.  
    - Inputs: Receives JSON data from "Get table data".  
    - Outputs: Passes documents to Text Splitter.  
    - Edge cases: Malformed JSON or missing fields.  
    - Version: 1

  - **Recursive Character Text Splitter**  
    - Type: LangChain Text Splitter Recursive Character Text Splitter  
    - Role: Splits large documents into chunks (500 characters with 100 overlap) for embedding generation.  
    - Inputs: Receives documents from Default Data Loader.  
    - Outputs: Passes chunks to Embeddings node.  
    - Version: 1

  - **Embeddings OpenAI for RAG**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates vector embeddings for text chunks.  
    - Inputs: Text chunks from Text Splitter.  
    - Outputs: Sends embeddings to Supabase Vector Store for insertion.  
    - Credentials: OpenAI API key required.  
    - Version: 1

  - **Supabase Vector Store to store vectors for RAG**  
    - Type: LangChain Vector Store Supabase (insert mode)  
    - Role: Inserts generated embeddings and documents metadata into Supabase for RAG search.  
    - Configuration:  
      - Table Name: "documents".  
      - Mode: Insert.  
    - Inputs: Receives embeddings and documents from Embeddings node and Default Data Loader.  
    - Credentials: Supabase API credentials required.  
    - Edge cases: Insert errors, duplicate data handling.  
    - Version: 1

#### 2.5 Supporting Information & Notes

- Sticky Notes are used throughout the workflow to provide context, setup instructions, and important reminders:

  - **Sticky Note5:** Introduction and setup instructions for using Peliqan, including trial signup, adding sources, running the RAG workflow, and editing AI Agent system messages.

  - **Sticky Note4:** Important environment variable reminder to enable tool usage with Peliqan SQL execution.

  - **Sticky Note ("Tools for AI Agent: SQL & RAG"):** Declares the purpose of SQL and RAG nodes as tools for the AI Agent.

  - **Sticky Note1:** Labels the AI Chatbot section.

  - **Sticky Note2:** Labels the "Brain" (AI Agent) section.

  - **Sticky Note3:** Labels the data feeding section for the vector store.

---

### 3. Summary Table

| Node Name                        | Node Type                               | Functional Role                       | Input Node(s)                  | Output Node(s)                            | Sticky Note                                                                                                        |
|---------------------------------|---------------------------------------|-------------------------------------|-------------------------------|------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| When chat message received       | LangChain Chat Trigger                 | Input reception / workflow trigger  | External webhook               | AI Agent                                | ## AI Chatbot                                                                                                      |
| AI Agent                        | LangChain Agent                       | Core AI logic and orchestration     | When chat message received, Simple Memory, OpenAI Chat Model | Execute an SQL query via Peliqan, Supabase Vector Store for search, OpenAI Chat Model output | ## Brain                                                                                                           |
| Simple Memory                   | LangChain Memory Buffer Window        | Maintains chat history context      | None                          | AI Agent                                | ## Brain                                                                                                           |
| OpenAI Chat Model               | LangChain LM Chat OpenAI               | Provides language model completions | AI Agent                      | AI Agent                                | ## Brain                                                                                                           |
| Execute an SQL query via Peliqan | Peliqan Tool Node                     | Executes SQL queries via Peliqan    | AI Agent                      | AI Agent                                | ⚠️ To use the Peliqan as tool with Operation "Execute SQL query": set env variable `N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=True` |
| Supabase Vector Store for search | LangChain Vector Store Supabase (retrieve-as-tool) | Provides semantic search over documents | AI Agent                      | AI Agent                                | ## Tools for AI Agent: SQL & RAG                                                                                   |
| Embeddings OpenAI for RAG retrieval | LangChain Embeddings OpenAI          | Generates embeddings for search     | Supabase Vector Store for search | Supabase Vector Store for search       | ## Tools for AI Agent: SQL & RAG                                                                                   |
| When clicking ‘Execute workflow’ | Manual Trigger                        | Manual start of data feeding block  | Manual trigger                | Get table data                          | ## Feed data into Vector Store (for RAG)                                                                           |
| Get table data                 | Peliqan Node                          | Retrieves business data for RAG     | When clicking ‘Execute workflow’ | Default Data Loader                    | ## Feed data into Vector Store (for RAG)                                                                           |
| Default Data Loader             | LangChain Document Default Data Loader | Converts raw JSON to documents      | Get table data                | Recursive Character Text Splitter       | ## Feed data into Vector Store (for RAG)                                                                           |
| Recursive Character Text Splitter | LangChain Text Splitter Recursive Character Text Splitter | Splits documents into chunks        | Default Data Loader           | Embeddings OpenAI for RAG               | ## Feed data into Vector Store (for RAG)                                                                           |
| Embeddings OpenAI for RAG       | LangChain Embeddings OpenAI           | Embeddings generation for data chunks | Recursive Character Text Splitter | Supabase Vector Store to store vectors for RAG | ## Feed data into Vector Store (for RAG)                                                                           |
| Supabase Vector Store to store vectors for RAG | LangChain Vector Store Supabase (insert mode) | Inserts embeddings and metadata     | Embeddings OpenAI for RAG, Default Data Loader | None                                   | ## Feed data into Vector Store (for RAG)                                                                           |
| Sticky Note5                   | Sticky Note                          | Introduction and setup instructions | None                          | None                                    | ## ℹ️ Introduction ... Visit peliqan.io/n8n for more information.                                                  |
| Sticky Note4                   | Sticky Note                          | Env variable reminder               | None                          | None                                    | ⚠️ To use the Peliqan as tool with Operation "Execute SQL query": set env variable `N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=True` |
| Sticky Note                    | Sticky Note                          | Label for SQL & RAG tools           | None                          | None                                    | ## Tools for AI Agent: SQL & RAG                                                                                   |
| Sticky Note1                  | Sticky Note                          | Label for AI Chatbot                 | None                          | None                                    | ## AI Chatbot                                                                                                      |
| Sticky Note2                  | Sticky Note                          | Label for Brain (AI Agent)           | None                          | None                                    | ## Brain                                                                                                           |
| Sticky Note3                  | Sticky Note                          | Label for data feeding into vector store | None                          | None                                    | ## Feed data into Vector Store (for RAG)                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Input Reception Node**  
   - Add a **LangChain Chat Trigger** node named **"When chat message received"**.  
   - Configure it as a public webhook to accept chat messages and trigger the workflow.

2. **Create the AI Agent Block**  
   - Add an **AI Agent** node named **"AI Agent"**.  
   - Configure system message with instructions about:  
     - Using vector store retrieval on Notion pages.  
     - Executing SQL queries on Peliqan using specified tables and columns schema (see system message content).  
   - Enable **Always Output Data**.  
   - Connect "When chat message received" node’s main output to the AI Agent’s main input.

3. **Configure Memory for AI Agent**  
   - Add a **Memory Buffer Window** node named **"Simple Memory"** with default settings.  
   - Connect its output to the AI Agent’s memory input.

4. **Configure Language Model for AI Agent**  
   - Add an **OpenAI Chat Model** node named **"OpenAI Chat Model"**.  
   - Attach your OpenAI API credentials.  
   - Connect its output to the AI Agent’s language model input.

5. **Configure AI Agent Tools**  
   - Create **Execute an SQL query via Peliqan** node (Peliqan Tool node):  
     - Set operation to execute SQL query.  
     - Configure dynamic SQL input from AI Agent variable `SQL_Query`.  
     - Connect the AI Agent tool output to this node.  
     - Attach Peliqan API credentials.  
     - **Important:** Set environment variable `N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=True` in your n8n environment to enable tool usage.  
   - Create **Supabase Vector Store for search** node:  
     - Set mode to "retrieve-as-tool".  
     - Provide table name "documents".  
     - Name tool "Company_knowledge_base".  
     - Add description for internal knowledge base use.  
     - Connect AI Agent tool output to this node.  
     - Attach Supabase API credentials.  
   - Create **Embeddings OpenAI for RAG retrieval** node:  
     - Attach OpenAI API credentials.  
     - Connect output to Supabase Vector Store for search node’s embeddings input.

6. **Connect AI Agent Tools back to AI Agent** (for output and chaining).

7. **Set Up Data Feeding for RAG Vector Store**

   - Add a **Manual Trigger** node named **"When clicking ‘Execute workflow’"** to start data loading manually.  

   - Add a **Peliqan Node** named **"Get table data"**:  
     - Set tableId to 200653 (or appropriate source table).  
     - Attach Peliqan API credentials.  
     - Connect manual trigger output to this node.  

   - Add **Default Data Loader** node:  
     - Configure to convert input JSON to document format with metadata fields `source_url` and `title`.  
     - Use expression for document content combining title and content fields.  
     - Connect "Get table data" output to this node.  

   - Add **Recursive Character Text Splitter** node:  
     - Configure chunk size = 500 characters, chunk overlap = 100 characters.  
     - Connect Default Data Loader output to this node.  

   - Add **Embeddings OpenAI for RAG** node:  
     - Attach OpenAI API credentials.  
     - Connect text splitter output to this node.  

   - Add **Supabase Vector Store to store vectors for RAG** node:  
     - Mode: Insert  
     - Table Name: "documents"  
     - Attach Supabase API credentials.  
     - Connect embeddings output and Default Data Loader output as inputs to this node.  

8. **Add Sticky Notes for Documentation** (optional but recommended for clarity):  
   - Provide introduction, setup instructions, environment variable reminders, and section labels as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| End-to-end demo of chatbot using business data from multiple sources with RAG + SQL. Peliqan.io is used as a data cache with ELT syncing. Setup requires Peliqan account, API key, and running RAG workflow to feed Supabase. Update AI Agent system message accordingly. | See Sticky Note5 content; https://peliqan.io/n8n   |
| To enable Peliqan SQL tool execution, set environment variable: `N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=True`                                                                                                                                                    | See Sticky Note4 content                           |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow and complies with all current content policies. It contains no illegal or offensive content. All processed data is legal and public.