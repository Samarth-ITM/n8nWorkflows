Create a Code Assistant that Learns from your GitHub Repository using OpenAI

https://n8nworkflows.xyz/workflows/create-a-code-assistant-that-learns-from-your-github-repository-using-openai-9993


# Create a Code Assistant that Learns from your GitHub Repository using OpenAI

---

### 1. Workflow Overview

This workflow, titled **"Create a Code Assistant that Learns from your GitHub Repository using OpenAI"**, is designed to build an AI-powered technical assistant specialized in answering developer questions by leveraging the source code and documentation of a specified GitHub repository.

It accomplishes this by:

- Synchronizing and loading source code files from a GitHub repository.
- Processing and splitting the source code into manageable chunks.
- Generating semantic embeddings using OpenAI embeddings for the code chunks.
- Storing these embeddings in an in-memory vector store to enable fast similarity searches.
- Responding to developer queries via a chat interface, where the AI agent uses the stored vector knowledge and memory buffers to provide contextually relevant answers.

The workflow logic can be grouped into three primary blocks:

- **1.1 Data Synchronization & Loading:** Fetching source files from GitHub, downloading, splitting, embedding, and storing them in a vector store.
- **1.2 AI Agent Setup:** Configuring the AI agent with chat capabilities, memory, and retrieval tools.
- **1.3 Chat Interaction:** Handling incoming chat messages, invoking the AI agent to answer based on the repository knowledge.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Data Synchronization & Loading

**Overview:**  
This block pulls source files from a GitHub repository, processes them into text chunks, generates embeddings using OpenAI, and stores them in an in-memory vector store. It updates the knowledge base for the AI agent.

**Nodes Involved:**  
- Sync Data (Manual Trigger)  
- Config (Set)  
- List files (GitHub)  
- Get File (HTTP Request)  
- Default Data Loader (LangChain Document Loader)  
- Recursive Character Text Splitter (LangChain Text Splitter)  
- Embeddings OpenAI (LangChain Embeddings)  
- Simple Vector Store1 (LangChain Vector Store In Memory)

**Node Details:**

- **Sync Data**  
  - Type: Manual Trigger  
  - Role: Entry point to start the sync process on demand.  
  - Configuration: Default; manual trigger node with no parameters.  
  - Input/Output: Outputs to `Config`.  
  - Failures: User must manually trigger; no automatic schedule.  

- **Config**  
  - Type: Set  
  - Role: Stores user-defined configuration parameters for the GitHub repo.  
  - Configuration: Four string variables:  
    - `repo_owner`: GitHub username or organization (default: "cphuong20202009")  
    - `repo_name`: Repository name (default: "share-n8n-workflow")  
    - `repo_path`: Path inside the repository (default: "share-n8n-workflow")  
    - `sub_path`: Subdirectory path for files (default: "workflows")  
  - Input: From `Sync Data`  
  - Output: To `List files`  
  - Failures: Incorrect config values will cause downstream API call failures.  

- **List files**  
  - Type: GitHub Node (List Files operation)  
  - Role: Lists files in the specified GitHub repository path.  
  - Configuration:  
    - Owner and repo name are dynamically set from `Config` node variables.  
    - File path set to `sub_path` from config.  
    - Operation: List files under that path.  
    - Credentials: Requires a GitHub API OAuth credential.  
  - Input: From `Config`  
  - Output: To `Get File`  
  - Failures: Authentication errors, rate limiting, invalid repo/path.  

- **Get File**  
  - Type: HTTP Request  
  - Role: Downloads the raw content of each file listed by the GitHub node.  
  - Configuration: URL set dynamically to `download_url` property of each file JSON.  
  - Input: From `List files`  
  - Output: To `Simple Vector Store1` (for insertion)  
  - Failures: HTTP errors if file is inaccessible or URL invalid.  

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Converts raw binary data (file content) into document format for LangChain processing.  
  - Configuration: Input data type is binary.  
  - Input: Not directly connected in this workflow (possibly for alternative data loading).  
  - Output: To `Simple Vector Store1` (via type `ai_document`)  
  - Failures: Data format mismatch, decoding errors.  

- **Recursive Character Text Splitter**  
  - Type: LangChain Text Splitter (Recursive Character)  
  - Role: Splits large documents into overlapping chunks for better embedding granularity.  
  - Configuration: Chunk overlap set to 100 characters.  
  - Input: Receives documents from `Default Data Loader`.  
  - Output: To `Default Data Loader` (feeding documents back)  
  - Failures: Chunking errors if input is empty or malformed.  

- **Embeddings OpenAI**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates semantic vector embeddings for each document chunk using OpenAI API.  
  - Configuration: Uses default OpenAI API model and parameters.  
  - Credentials: Requires valid OpenAI API key.  
  - Input: Connected to `Simple Vector Store1` (for embedding generation)  
  - Output: To `Simple Vector Store1` (embedding insertion)  
  - Failures: API rate limits, authentication errors, network timeouts.  

- **Simple Vector Store1**  
  - Type: LangChain Vector Store In Memory (Insert mode)  
  - Role: Stores embeddings and associated metadata in memory for fast retrieval.  
  - Configuration: Uses memory key named `"source-code"` to identify stored vectors.  
  - Input: Receives embedded vectors from `Embeddings OpenAI` and document data from `Get File`.  
  - Output: None (terminal for sync data flow)  
  - Failures: Memory overflow if too large dataset; restart clears store.  

---

#### 1.2 AI Agent Setup

**Overview:**  
This block configures the AI agent which handles developer queries. It includes memory buffers, language models, vector store tools, and chat models, enabling the agent to interactively answer questions with context from the synced codebase.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- Vector Store Tool (LangChain Tool for vector retrieval)  
- Window Buffer Memory (LangChain Memory Buffer)  
- Simple Vector Store (LangChain Vector Store In Memory)  
- Embeddings OpenAI1 (LangChain Embeddings)  
- OpenAI Chat Model (LangChain Chat Model)  
- OpenAI Chat Model1 (LangChain Chat Model)

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Central AI assistant that processes queries, uses tools, memory, and language models to formulate answers.  
  - Configuration: System message instructs it to act as a technical assistant for developer questions, explicitly referencing a tool named `project_source_tool` to retrieve codebase info.  
  - Inputs: Receives chat messages, memory state, and tools.  
  - Outputs: Answers to chat interface.  
  - Failures: Model invocation errors, tool integration failures, memory issues.  

- **Vector Store Tool**  
  - Type: LangChain Tool (Vector Store)  
  - Role: Tool connected to the vector store that retrieves top K relevant documents based on query similarity.  
  - Configuration:  
    - Name: `project_source_tool` (matches AI Agent tool reference)  
    - TopK: 5 (returns top 5 relevant documents)  
    - Description: "Retrieve information from any source code"  
  - Inputs: Receives queries from AI Agent.  
  - Outputs: Returns relevant document snippets to AI Agent.  
  - Failures: Vector store lookup errors, empty results.  

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer (Window)  
  - Role: Keeps a sliding window of recent chat interactions to provide conversational context.  
  - Configuration: Default window size (not explicitly set here).  
  - Inputs: Receives chat and agent outputs.  
  - Outputs: Feeds memory context to AI Agent.  
  - Failures: Memory overflow or loss on restart (in-memory).  

- **Simple Vector Store**  
  - Type: LangChain Vector Store In Memory (Read mode)  
  - Role: Provides the vector store instance used by `Vector Store Tool` for search.  
  - Configuration: Uses memory key `"source-code"`, consistent with the data sync block.  
  - Inputs: Receives embeddings from `Embeddings OpenAI1`.  
  - Outputs: Feeds vector data to `Vector Store Tool`.  
  - Failures: Empty or stale vector data if sync not done.  

- **Embeddings OpenAI1**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates embeddings for input queries or documents (used in retrieval).  
  - Configuration: Same OpenAI API credentials as embeddings in sync block.  
  - Inputs: Feeds embeddings into `Simple Vector Store`.  
  - Failures: API failures as before.  

- **OpenAI Chat Model**  
  - Type: LangChain Chat Model using OpenAI  
  - Role: Language model used by the AI Agent to generate responses.  
  - Configuration: Model name `"gpt-4.1-mini"` (a smaller GPT-4 variant), default parameters.  
  - Inputs: Receives prompt from AI Agent.  
  - Outputs: Provides natural language answers.  
  - Failures: API errors, rate limits, model availability.  

- **OpenAI Chat Model1**  
  - Type: LangChain Chat Model using OpenAI  
  - Role: Additional Chat model instance connected directly to AI Agent for possibly multi-step or fallback generation.  
  - Configuration: Same as `OpenAI Chat Model`.  
  - Inputs/Outputs: Connects to AI Agent.  
  - Failures: Same as above.  

---

#### 1.3 Chat Interaction

**Overview:**  
This block listens for incoming chat messages from users, triggers the AI agent to process those messages using the knowledge base and memory, and returns contextually relevant answers.

**Nodes Involved:**  
- When chat message received (LangChain Chat Trigger)  
- AI Agent (LangChain Agent)

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Webhook node that listens for incoming chat messages from external clients (e.g., UI or chatbot interface).  
  - Configuration: Uses webhook ID to receive messages, no special options configured.  
  - Input: External chat message.  
  - Output: Sends messages to `AI Agent`.  
  - Failures: Webhook connectivity issues, malformed input.  

- **AI Agent**  
  - Same node as described in block 1.2; here it acts on live chat input.  

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                               | Input Node(s)               | Output Node(s)                | Sticky Note                                                       |
|---------------------------|---------------------------------------------|-----------------------------------------------|-----------------------------|------------------------------|------------------------------------------------------------------|
| Sync Data                 | Manual Trigger                              | Starts the GitHub data sync process           |                             | Config                       |                                                                  |
| Config                    | Set                                         | Stores GitHub repo configuration parameters   | Sync Data                   | List files                   | See Sticky Note1 for setup guide                                 |
| List files                | GitHub                                      | Lists files in specified repo path             | Config                      | Get File                     |                                                                  |
| Get File                  | HTTP Request                                | Downloads raw content of each file             | List files                  | Simple Vector Store1         |                                                                  |
| Default Data Loader       | LangChain Document Loader                    | Converts raw file data into documents          | Recursive Character Text Splitter (via AI doc) | Simple Vector Store1         |                                                                  |
| Recursive Character Text Splitter | LangChain Text Splitter               | Splits documents into overlapping chunks       | Default Data Loader          | Default Data Loader          |                                                                  |
| Embeddings OpenAI         | LangChain Embeddings                         | Generates embeddings for document chunks       | Simple Vector Store1 (embedding input) | Simple Vector Store1 (embedding output) |                                                                  |
| Simple Vector Store1      | LangChain Vector Store In Memory (Insert)  | Stores embeddings and documents in memory      | Get File, Embeddings OpenAI  |                              |                                                                  |
| AI Agent                  | LangChain Agent                             | Core AI assistant processing queries           | When chat message received, Window Buffer Memory, Vector Store Tool, OpenAI Chat Model1 |                              | See Sticky Note2 for AI usage                                    |
| Vector Store Tool         | LangChain Tool (Vector Store retrieval)     | Retrieves relevant documents from vector store | AI Agent                    | AI Agent                     |                                                                  |
| Window Buffer Memory      | LangChain Memory Buffer                      | Stores recent conversation context             | AI Agent                    | AI Agent                     |                                                                  |
| Simple Vector Store       | LangChain Vector Store In Memory (Read)    | Provides vector store data for retrieval       | Embeddings OpenAI1          | Vector Store Tool            |                                                                  |
| Embeddings OpenAI1        | LangChain Embeddings                         | Embeddings generator for query/documents       | Simple Vector Store         |                              |                                                                  |
| OpenAI Chat Model         | LangChain Chat Model                         | Language model generating text responses       | Vector Store Tool           | Vector Store Tool            |                                                                  |
| OpenAI Chat Model1        | LangChain Chat Model                         | Language model used by AI Agent                 | AI Agent                    | AI Agent                     |                                                                  |
| When chat message received| LangChain Chat Trigger                      | Entry point for receiving user queries         |                             | AI Agent                     |                                                                  |
| Sticky Note1              | Sticky Note                                 | Setup guide instructions                        |                             |                              | ## Setup Guide: Update Config node, connect GitHub account, trigger Sync Data |
| Sticky Note2              | Sticky Note                                 | Usage instructions for chat interface           |                             |                              | Ask questions to the AI Agent — it will respond using your repository knowledge. |
| Sticky Note               | Sticky Note                                 | Reminder to update knowledge base               |                             |                              | Pull your source files and update the knowledge base (vectorstore) for the AI Agent |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Sync Data`  
   - Purpose: To manually initiate the GitHub sync process.  

2. **Create Set Node**  
   - Name: `Config`  
   - Purpose: Store repository parameters.  
   - Add string fields:  
     - `repo_owner` (e.g., "cphuong20202009")  
     - `repo_name` (e.g., "share-n8n-workflow")  
     - `repo_path` (e.g., "share-n8n-workflow")  
     - `sub_path` (e.g., "workflows")  
   - Connect `Sync Data` → `Config`.  

3. **Create GitHub Node**  
   - Name: `List files`  
   - Operation: List files  
   - Repository: Use `repo_name` from `Config`  
   - Owner: Use `repo_owner` from `Config`  
   - File Path: Use `sub_path` from `Config`  
   - Credentials: Connect your GitHub OAuth credential  
   - Connect `Config` → `List files`.  

4. **Create HTTP Request Node**  
   - Name: `Get File`  
   - Set URL to dynamically use `{{$json["download_url"]}}` from `List files` output.  
   - Connect `List files` → `Get File`.  

5. **Create LangChain Nodes for Document Processing:**  
   - `Default Data Loader` (Type: documentDefaultDataLoader)  
     - Configure to accept binary data.  
   - `Recursive Character Text Splitter` (Type: textSplitterRecursiveCharacterTextSplitter)  
     - Set chunk overlap to 100 characters.  
   - Connect `Get File` → `Default Data Loader` → `Recursive Character Text Splitter` → `Default Data Loader` (loop for processing).  

6. **Create Embeddings Node**  
   - Name: `Embeddings OpenAI`  
   - Use OpenAI API credentials.  
   - Connect `Default Data Loader` output to `Embeddings OpenAI`.  

7. **Create Vector Store Node (Insert Mode)**  
   - Name: `Simple Vector Store1`  
   - Set mode to "insert".  
   - Use memory key `"source-code"` for vector storage.  
   - Connect `Embeddings OpenAI` → `Simple Vector Store1`.  
   - Also connect `Get File` (raw content) → `Simple Vector Store1` (document input).  

8. **Create Vector Store Node (Read Mode)**  
   - Name: `Simple Vector Store`  
   - Use memory key `"source-code"` consistent with insert node.  
   - Set mode to "read".  

9. **Create Embeddings Node for Queries**  
   - Name: `Embeddings OpenAI1`  
   - Use same OpenAI credentials.  
   - Connect `Simple Vector Store` → `Embeddings OpenAI1`.  

10. **Create Vector Store Tool Node**  
    - Name: `Vector Store Tool`  
    - Tool name: `project_source_tool` (must match AI Agent tool reference)  
    - TopK: 5  
    - Description: "Retrieve information from any source code"  
    - Connect `Simple Vector Store` → `Vector Store Tool`.  

11. **Create Window Buffer Memory Node**  
    - Name: `Window Buffer Memory`  
    - Default sliding window for chat memory context.  

12. **Create OpenAI Chat Model Nodes**  
    - Name: `OpenAI Chat Model` and `OpenAI Chat Model1`  
    - Model: `gpt-4.1-mini` or similar GPT-4 variant  
    - Use OpenAI credentials.  

13. **Create AI Agent Node**  
    - Name: `AI Agent`  
    - System message:  
      ```
      You are a helpful technical assistant designed to answer developer questions based on the project’s source code and technical documentation.

      When a developer asks a question, use the tool `project_source_tool` to retrieve relevant information from the available codebase and documentation.
      ```  
    - Connect:  
      - Inputs:  
        - From `When chat message received` (main)  
        - From `Window Buffer Memory` (memory)  
        - From `Vector Store Tool` (tool)  
        - From `OpenAI Chat Model1` (language model)  
      - Outputs: To chat interface (not explicitly shown)  

14. **Create Chat Trigger Node**  
    - Name: `When chat message received`  
    - Type: Chat Trigger (webhook)  
    - Connect output to `AI Agent`.  

15. **Connect Nodes According to the Workflow Connections:**  
    - `Sync Data` → `Config` → `List files` → `Get File` → `Simple Vector Store1`  
    - `Simple Vector Store1` → `Embeddings OpenAI` → `Simple Vector Store1` (embedding input)  
    - `Simple Vector Store` → `Vector Store Tool` → `AI Agent`  
    - `Window Buffer Memory` → `AI Agent`  
    - `OpenAI Chat Model1` → `AI Agent`  
    - `When chat message received` → `AI Agent`  

16. **Credential Setup:**  
    - GitHub OAuth account with repo read access.  
    - OpenAI API key with sufficient quota and access to GPT-4.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Setup Guide: Update the `Config` node with your repository details, connect your GitHub account, and trigger “Sync Data” to load your repository into the AI knowledge base. | Refer to Sticky Note1 in the workflow for detailed instructions.     |
| After syncing data, ask questions to the AI Agent; it will answer using your repository knowledge. | Refer to Sticky Note2 in the workflow for usage instructions.        |
| The vector store is in-memory and will reset if the workflow or n8n instance restarts.         | Important to re-sync after restart for persistent knowledge.          |
| Model used is `gpt-4.1-mini` - ensure your OpenAI API key has access to GPT-4 models.          | OpenAI model naming conventions may change; verify model availability.|
| For detailed LangChain nodes documentation visit: https://docs.n8n.io/integrations/builtin/nodes/langchain/ | Useful for customizing or extending LangChain functionality.          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.