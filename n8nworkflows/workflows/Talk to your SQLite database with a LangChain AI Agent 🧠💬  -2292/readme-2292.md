Talk to your SQLite database with a LangChain AI Agent 🧠💬  

https://n8nworkflows.xyz/workflows/talk-to-your-sqlite-database-with-a-langchain-ai-agent--------2292


# Talk to your SQLite database with a LangChain AI Agent 🧠💬  

### 1. Workflow Overview

This n8n workflow demonstrates an AI-powered agent that interacts with a local SQLite database (`chinook.db`) using natural language queries. It leverages LangChain’s SQL agent capabilities combined with OpenAI’s GPT-4 Turbo model to understand user input, generate SQL queries, and provide context-aware responses based on database content.

The workflow is organized into three main logical blocks:

- **1.1 Initial Setup (Database Download and Extraction)**  
  Downloads a sample SQLite database in ZIP form from an external URL, extracts it, and saves the database file locally. This step is designed to be run once before engaging the chat interaction.

- **1.2 Chat Input Reception and Data Preparation**  
  Waits for chat messages from users, loads the locally saved SQLite database (binary), and combines the chat input JSON with the database binary data to prepare for AI processing.

- **1.3 AI Agent Processing with Memory and OpenAI Integration**  
  Uses a LangChain SQL agent configured with OpenAI’s GPT-4 Turbo model and a window buffer memory to process the combined chat input and SQLite data. The agent generates SQL queries as needed, retrieves data, and returns answers. The conversation context is preserved in memory for ongoing dialogue coherence.

---

### 2. Block-by-Block Analysis

#### 2.1 Initial Setup (Database Download and Extraction)

- **Overview:**  
  Downloads the example SQLite database ZIP archive, extracts it to obtain the `chinook.db` file, and saves this file locally on the system. This block is intended to be executed once during setup.

- **Nodes Involved:**  
  - When clicking "Test workflow"  
  - Get chinook.zip example  
  - Extract zip file  
  - Save chinook.db locally  
  - Sticky Note (instructions)

- **Node Details:**

  - **When clicking "Test workflow"**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually for setup purposes.  
    - Config: No parameters; simply triggers downstream nodes.  
    - Connections: Outputs to `Get chinook.zip example`.  
    - Failures: None expected; manual trigger.

  - **Get chinook.zip example**  
    - Type: HTTP Request  
    - Role: Downloads the ZIP archive containing the sample SQLite database from https://www.sqlitetutorial.net/wp-content/uploads/2018/03/chinook.zip.  
    - Config: Uses default HTTP GET; no authentication or special headers.  
    - Output: Binary ZIP file data to `Extract zip file`.  
    - Failures: Network errors, URL not reachable, download timeout.

  - **Extract zip file**  
    - Type: Compression (Extract)  
    - Role: Extracts ZIP archive binary data received from the HTTP Request node.  
    - Config: Default extraction settings; assumes single file archive.  
    - Output: Binary data of extracted `chinook.db` file to `Save chinook.db locally`.  
    - Failures: Corrupted ZIP file, extraction errors.

  - **Save chinook.db locally**  
    - Type: Read/Write File  
    - Role: Writes the extracted SQLite database file to local storage as `./chinook.db`.  
    - Config: Write operation, file name explicitly set, binary data source is the extracted file.  
    - Failures: File system permissions, disk space, invalid path.

  - **Sticky Note**  
    - Purpose: Provides user instructions about this setup block, emphasizing it runs once and pointing to the database source URL.  
    - Content Summary: Explains download, extraction, saving steps and readiness for chat interactions.

---

#### 2.2 Chat Input Reception and Data Preparation

- **Overview:**  
  This block waits for user chat inputs, loads the locally saved SQLite database file, and merges the chat message JSON with the binary database data to prepare input for the AI agent.

- **Nodes Involved:**  
  - Chat Trigger  
  - Load local chinook.db  
  - Combine chat input with the binary  
  - Sticky Note1 (instructions)

- **Node Details:**

  - **Chat Trigger**  
    - Type: LangChain Chat Trigger  
    - Role: Receives chat messages from users via webhook.  
    - Config: Default webhook settings; webhook ID configured.  
    - Output: JSON chat input to `Load local chinook.db`.  
    - Failures: Webhook connectivity, unauthorized access.

  - **Load local chinook.db**  
    - Type: Read/Write File  
    - Role: Reads the locally saved SQLite database file (`./chinook.db`) from disk.  
    - Config: Read operation with file selector set to `./chinook.db`.  
    - Output: Binary SQLite data to `Combine chat input with the binary`.  
    - Failures: File missing, read permission errors.

  - **Combine chat input with the binary**  
    - Type: Set  
    - Role: Combines incoming chat JSON and the SQLite binary data into a single JSON object that includes both text input and database content.  
    - Config: Raw mode with `includeBinary` enabled; JSON output set to the previous chat trigger’s JSON data.  
    - Output: Combined JSON + binary to `AI Agent`.  
    - Failures: Expression evaluation errors, missing binary data.

  - **Sticky Note1**  
    - Purpose: Explains that for each chat message, the SQLite database is loaded and combined with user input.  
    - Content Summary: Highlights the data preparation step for AI processing.

---

#### 2.3 AI Agent Processing with Memory and OpenAI Integration

- **Overview:**  
  Processes the combined chat input and SQLite database through a LangChain SQL agent using OpenAI GPT-4 Turbo. It uses window buffer memory to maintain conversation context and generates SQL queries dynamically to answer user questions.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Window Buffer Memory  
  - AI Agent  
  - Sticky Note2 (instructions)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides the language model backend (GPT-4 Turbo) used by the AI agent for natural language understanding and response generation.  
    - Config: Model set to "gpt-4-turbo", temperature 0.3 for controlled randomness.  
    - Credentials: Uses stored OpenAI API credentials named "Ted's Tech Talks OpenAi".  
    - Output: Passes language model responses to `AI Agent`.  
    - Failures: API authentication errors, rate limits, timeout errors.

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a sliding window of the last 10 interactions to preserve chat context.  
    - Config: `contextWindowLength` set to 10, storing last 10 messages for memory recall.  
    - Output: Provides memory context to `AI Agent`.  
    - Failures: Memory overflow unlikely; misconfiguration could cause context loss.

  - **AI Agent**  
    - Type: LangChain Agent (SQL Agent)  
    - Role: Core processing node that orchestrates the LangChain SQL agent. It receives combined chat input and database binary, coordinates with OpenAI and memory nodes, executes SQL queries against the SQLite data, and generates final answers.  
    - Config: Agent type set to "sqlAgent", data source set to "sqlite", no extra options configured.  
    - Inputs: Receives inputs from `Combine chat input with the binary`, `OpenAI Chat Model` (language model), and `Window Buffer Memory` (memory).  
    - Outputs: Final AI response with SQL-derived answers.  
    - Failures: SQL execution errors, malformed queries, OpenAI API issues, memory context errors.

  - **Sticky Note2**  
    - Purpose: Explains the behavior of the LangChain SQL Agent making multiple queries if needed and storing answers in memory for conversational continuity.  
    - Content Summary: Example queries and explanation of agent iterative processing.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                | Input Node(s)                | Output Node(s)                  | Sticky Note                                   |
|---------------------------|----------------------------------|------------------------------------------------|------------------------------|--------------------------------|-----------------------------------------------|
| When clicking "Test workflow" | Manual Trigger                  | Initiates the initial database download setup | None                         | Get chinook.zip example          | ## Run this part only once ...                |
| Get chinook.zip example    | HTTP Request                     | Downloads ZIP archive with SQLite database     | When clicking "Test workflow" | Extract zip file                | ## Run this part only once ...                |
| Extract zip file           | Compression (Extract)             | Extracts the SQLite DB file from ZIP            | Get chinook.zip example       | Save chinook.db locally          | ## Run this part only once ...                |
| Save chinook.db locally    | Read/Write File                  | Saves extracted SQLite DB to local disk         | Extract zip file              | None                           | ## Run this part only once ...                |
| Chat Trigger              | LangChain Chat Trigger            | Receives chat input via webhook                  | None                         | Load local chinook.db            | ## On every chat message ...                   |
| Load local chinook.db      | Read/Write File                  | Loads saved SQLite DB from local disk            | Chat Trigger                 | Combine chat input with the binary | ## On every chat message ...                   |
| Combine chat input with the binary | Set                       | Combines chat JSON input with SQLite binary data| Load local chinook.db         | AI Agent                      | ## On every chat message ...                   |
| OpenAI Chat Model          | LangChain LM Chat OpenAI          | Provides GPT-4 Turbo language model for agent   | None (credential node)         | AI Agent                      |                                               |
| Window Buffer Memory       | LangChain Memory Buffer Window    | Maintains last 10 message conversation context  | None (memory node)             | AI Agent                      |                                               |
| AI Agent                  | LangChain Agent (SQL Agent)       | Processes input, runs SQL queries, generates response | Combine chat input with the binary, OpenAI Chat Model, Window Buffer Memory | None                      | ### LangChain SQL Agent can make several queries before producing the final answer... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking "Test workflow"`  
   - Purpose: To initiate the database download and extraction process.

2. **Create an HTTP Request node**  
   - Name: `Get chinook.zip example`  
   - URL: `https://www.sqlitetutorial.net/wp-content/uploads/2018/03/chinook.zip`  
   - Method: GET  
   - Connect output of `When clicking "Test workflow"` to this node.

3. **Create a Compression node**  
   - Name: `Extract zip file`  
   - Operation: Extract ZIP archive (default settings)  
   - Connect output of `Get chinook.zip example` to this node.

4. **Create a Read/Write File node**  
   - Name: `Save chinook.db locally`  
   - Operation: Write  
   - File Name: `./chinook.db`  
   - Data Source: Use binary data from `Extract zip file`  
   - Connect output of `Extract zip file` to this node.

5. **Create a LangChain Chat Trigger node**  
   - Name: `Chat Trigger`  
   - Setup webhook to receive chat messages.  
   - This acts as the entry point for user chat messages.

6. **Create a Read/Write File node**  
   - Name: `Load local chinook.db`  
   - Operation: Read  
   - File Selector: `./chinook.db`  
   - Connect output of `Chat Trigger` to this node.

7. **Create a Set node**  
   - Name: `Combine chat input with the binary`  
   - Mode: Raw  
   - Enable "Include Binary"  
   - JSON output: Set to the JSON data from `Chat Trigger` (expression: `={{ $('Chat Trigger').item.json }}`)  
   - Connect output of `Load local chinook.db` to this node.

8. **Create a LangChain LM Chat OpenAI node**  
   - Name: `OpenAI Chat Model`  
   - Model: `gpt-4-turbo`  
   - Options: Temperature = 0.3  
   - Credentials: Configure with valid OpenAI API credentials.  
   - No direct input connection; this is a resource node.

9. **Create a LangChain Memory Buffer Window node**  
   - Name: `Window Buffer Memory`  
   - Context Window Length: 10  
   - No direct input connection; this is a resource node.

10. **Create a LangChain Agent node**  
    - Name: `AI Agent`  
    - Agent: `sqlAgent` (LangChain SQL Agent)  
    - Data Source: `sqlite`  
    - Connect output of `Combine chat input with the binary` to this node's main input.  
    - Connect `OpenAI Chat Model` node to the AI Agent’s language model input.  
    - Connect `Window Buffer Memory` node to the AI Agent’s memory input.

**Notes on Credentials:**  
- OpenAI API credentials must be created and assigned to `OpenAI Chat Model`.  
- No special credentials are needed for local file operations.

**Workflow Execution:**  
- Run the `When clicking "Test workflow"` node once to download and save the SQLite DB locally.  
- Then start sending chat messages to the webhook URL generated by `Chat Trigger`.  
- The AI Agent will process each message in the context of the SQLite database and memory.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                   |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Full article detailing this AI agent with SQLite example is available at https://blog.n8n.io/ai-agents/                        | Official n8n blog article on AI Agents           |
| The SQLite example database is from https://www.sqlitetutorial.net/sqlite-sample-database/                                     | Source of chinook.db sample database              |
| The LangChain SQL Agent supports multi-step query generation, enabling complex questions like "What are the revenues by genre?" | Sticky note explanation in workflow               |
| Temperature setting of 0.3 on GPT-4 Turbo balances creativity and factual accuracy for SQL queries and answers                | Configuration recommendation                       |

---

This document provides a comprehensive reference for understanding, reproducing, and maintaining the "SQL agent with memory" n8n workflow that integrates LangChain AI and SQLite databases for conversational data querying.