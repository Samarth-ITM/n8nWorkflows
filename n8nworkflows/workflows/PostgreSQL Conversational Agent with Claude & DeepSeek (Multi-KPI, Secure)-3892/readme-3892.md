PostgreSQL Conversational Agent with Claude & DeepSeek (Multi-KPI, Secure)

https://n8nworkflows.xyz/workflows/postgresql-conversational-agent-with-claude---deepseek--multi-kpi--secure--3892


# PostgreSQL Conversational Agent with Claude & DeepSeek (Multi-KPI, Secure)

### 1. Workflow Overview

This workflow implements a secure, AI-driven conversational agent for querying PostgreSQL databases via natural language, using n8n’s Model Context Protocol (MCP). It enables users to ask multi-KPI questions in a single message and receive consolidated, human-readable reports without exposing raw SQL or requiring visual outputs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Triggering:** Handles incoming user queries via MCP and chat triggers.
- **1.2 AI Agent & Memory:** Processes user input, manages conversation context, and plans multi-step query execution.
- **1.3 Database Schema Exploration:** Retrieves database schema and table lists dynamically for secure query generation.
- **1.4 SQL Generation & Execution:** Generates parameterized SQL queries via a subworkflow and executes them safely.
- **1.5 CRUD Operations Dispatcher:** Routes database operations (read, create, update) based on AI agent instructions.
- **1.6 Subworkflow Integration:** Invokes a utility subworkflow for SQL validation and formatting.
- **1.7 Model & Tool Configuration:** Uses Claude 3.5 Haiku for conversational AI and DeepSeek for SQL generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering

**Overview:**  
Receives user queries either through the MCP Server Trigger endpoint or a chat message trigger, initiating the workflow.

**Nodes Involved:**  
- PostgreSQL MCP Server  
- When chat message received  
- When Executed by Another Workflow

**Node Details:**

- **PostgreSQL MCP Server**  
  - Type: MCP Trigger  
  - Role: Listens on `/mcp/...` endpoint for incoming user queries formatted per MCP standard.  
  - Configuration: Default webhook ID, no special parameters.  
  - Inputs: External HTTP requests.  
  - Outputs: Sends data to AI Agent and schema exploration tools.  
  - Edge Cases: Webhook misconfiguration, network errors, malformed MCP messages.

- **When chat message received**  
  - Type: Chat Trigger  
  - Role: Alternative trigger for chat-based input, supporting conversational UI integration.  
  - Configuration: Default webhook ID, no parameters.  
  - Inputs: Incoming chat messages.  
  - Outputs: Forwards messages to AI Agent.  
  - Edge Cases: Message format errors, webhook downtime.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be invoked as a subworkflow or utility from other workflows.  
  - Configuration: No parameters.  
  - Inputs: Triggered execution calls.  
  - Outputs: Routes to CRUD operation dispatcher.  
  - Edge Cases: Invalid execution context, missing parameters.

---

#### 2.2 AI Agent & Memory

**Overview:**  
Manages conversation context, interprets user intent, plans multi-step query execution, and maintains session memory.

**Nodes Involved:**  
- AI Agent  
- Simple Memory  
- Anthropic Chat Model  
- MCP Client  
- Think

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Core reasoning and planning engine, orchestrates multi-KPI query generation and tool usage.  
  - Configuration: Uses Anthropic Chat Model as language model, connected to Simple Memory and MCP Client tools.  
  - Inputs: User queries from triggers.  
  - Outputs: Sends planned operations to CRUD dispatcher and Think tool.  
  - Edge Cases: Model API errors, token limits, unexpected input formats.

- **Simple Memory**  
  - Type: Memory Buffer Window  
  - Role: Stores recent conversation history to maintain context across interactions.  
  - Configuration: Default window size, no custom parameters.  
  - Inputs: Conversation data from AI Agent.  
  - Outputs: Provides context back to AI Agent.  
  - Edge Cases: Memory overflow, context loss.

- **Anthropic Chat Model**  
  - Type: Language Model (Anthropic Claude 3.5 Haiku)  
  - Role: Provides natural language understanding and generation capabilities for the AI Agent.  
  - Configuration: Uses Anthropic API credentials, optimized for MCP compatibility.  
  - Inputs: Prompts from AI Agent.  
  - Outputs: Generated text responses.  
  - Edge Cases: API rate limits, authentication failures.

- **MCP Client**  
  - Type: MCP Client Tool  
  - Role: Facilitates communication between AI Agent and MCP Server Trigger, enabling tool calls and data exchange.  
  - Configuration: Default.  
  - Inputs: AI Agent requests.  
  - Outputs: MCP Server Trigger.  
  - Edge Cases: Communication failures, protocol mismatches.

- **Think**  
  - Type: Tool Think  
  - Role: Breaks down complex user queries into structured goals or subtasks for the AI Agent.  
  - Configuration: Default.  
  - Inputs: AI Agent output.  
  - Outputs: Refined tasks back to AI Agent.  
  - Edge Cases: Misinterpretation of intent, incomplete breakdown.

---

#### 2.3 Database Schema Exploration

**Overview:**  
Dynamically retrieves database schema and table lists to inform secure SQL generation and prevent raw SQL injection.

**Nodes Involved:**  
- ListTables  
- GetTableSchema  
- get table details (toolWorkflow)  
- ReadTableRows (toolWorkflow)  
- CreateTableRecords (toolWorkflow)  
- UpdateTableRecords (toolWorkflow)

**Node Details:**

- **ListTables**  
  - Type: Postgres Tool  
  - Role: Retrieves list of tables from the connected PostgreSQL database.  
  - Configuration: Uses PostgreSQL credentials set in n8n.  
  - Inputs: Triggered by MCP Server.  
  - Outputs: Table list to AI Agent for query planning.  
  - Edge Cases: DB connection failures, permission errors.

- **GetTableSchema**  
  - Type: Postgres Tool  
  - Role: Retrieves schema details (columns, types) for specified tables.  
  - Configuration: Uses PostgreSQL credentials.  
  - Inputs: Triggered by MCP Server.  
  - Outputs: Schema details to AI Agent and SQL generation tools.  
  - Edge Cases: Schema changes during runtime, permission issues.

- **get table details**  
  - Type: Tool Workflow (LangChain)  
  - Role: Subworkflow that processes schema info to provide structured table metadata.  
  - Configuration: Uses DeepSeek model for structured output.  
  - Inputs: From MCP Server.  
  - Outputs: Structured table metadata to AI Agent.  
  - Edge Cases: Model parsing errors, incomplete schema data.

- **ReadTableRows**  
  - Type: Tool Workflow (LangChain)  
  - Role: Reads rows from tables based on AI-generated queries.  
  - Configuration: Uses DeepSeek for SQL generation and safe execution.  
  - Inputs: MCP Server.  
  - Outputs: Query results to AI Agent.  
  - Edge Cases: Query timeouts, empty results.

- **CreateTableRecords**  
  - Type: Tool Workflow (LangChain)  
  - Role: Handles batch insertion of records securely.  
  - Configuration: Uses parameterized queries.  
  - Inputs: MCP Server.  
  - Outputs: Confirmation to AI Agent.  
  - Edge Cases: Constraint violations, partial failures.

- **UpdateTableRecords**  
  - Type: Tool Workflow (LangChain)  
  - Role: Handles batch updates securely.  
  - Configuration: Parameterized updates only.  
  - Inputs: MCP Server.  
  - Outputs: Confirmation to AI Agent.  
  - Edge Cases: Conflicts, concurrency issues.

---

#### 2.4 SQL Generation & Execution

**Overview:**  
Generates parameterized SQL queries from natural language using DeepSeek-powered subworkflow and executes them securely without raw SQL exposure.

**Nodes Involved:**  
- get_query_and_data (referenced in description, implemented as subworkflow invoked within tool workflows)  
- checkdatabase (external subworkflow, uploaded separately)

**Node Details:**

- **get_query_and_data**  
  - Type: Subworkflow (not explicitly in main workflow JSON, but referenced)  
  - Role: Converts natural language KPI requests into parameterized SQL queries using DeepSeek.  
  - Configuration: Uses DeepSeek model credentials.  
  - Inputs: Structured goals from AI Agent.  
  - Outputs: SQL queries and parameters for execution.  
  - Edge Cases: Incorrect SQL generation, model hallucination.

- **checkdatabase**  
  - Type: External Subworkflow (uploaded separately)  
  - Role: Validates generated SQL, executes queries, formats results as clean text.  
  - Configuration: PostgreSQL credentials, formatting rules.  
  - Inputs: SQL queries from get_query_and_data.  
  - Outputs: Human-readable query results.  
  - Edge Cases: SQL errors, formatting failures.

---

#### 2.5 CRUD Operations Dispatcher

**Overview:**  
Routes database operations based on AI Agent instructions to appropriate PostgreSQL nodes for reading, creating, or updating records.

**Nodes Involved:**  
- Operation (Switch)  
- ReadTableRecord (Postgres)  
- CreateTableRecord (Postgres)  
- UpdateTableRecord (Postgres)

**Node Details:**

- **Operation**  
  - Type: Switch  
  - Role: Determines which CRUD operation to perform based on AI Agent’s command.  
  - Configuration: Switches on operation type (read, create, update).  
  - Inputs: Triggered by "When Executed by Another Workflow".  
  - Outputs: Routes to corresponding Postgres node.  
  - Edge Cases: Unknown operation types, missing parameters.

- **ReadTableRecord**  
  - Type: Postgres  
  - Role: Executes SELECT queries to read records.  
  - Configuration: Parameterized queries, always outputs data.  
  - Inputs: From Operation node.  
  - Outputs: Query results.  
  - Edge Cases: No matching records, permission issues.

- **CreateTableRecord**  
  - Type: Postgres  
  - Role: Inserts a single record into a table.  
  - Configuration: Parameterized INSERT statements.  
  - Inputs: From Operation node.  
  - Outputs: Insert confirmation.  
  - Edge Cases: Constraint violations.

- **UpdateTableRecord**  
  - Type: Postgres  
  - Role: Updates a single record in a table.  
  - Configuration: Parameterized UPDATE statements.  
  - Inputs: From Operation node.  
  - Outputs: Update confirmation.  
  - Edge Cases: Record not found, concurrency conflicts.

---

#### 2.6 Subworkflow Integration

**Overview:**  
Invokes external subworkflows for SQL generation, validation, and formatting to maintain modularity and security.

**Nodes Involved:**  
- CreateTableRecords (toolWorkflow)  
- UpdateTableRecords (toolWorkflow)  
- get table details (toolWorkflow)  
- ReadTableRows (toolWorkflow)

**Node Details:**

- These nodes invoke subworkflows that handle specific database operations or metadata retrieval using DeepSeek and other models.  
- They ensure all SQL is generated and executed securely, without raw SQL exposure.  
- Inputs and outputs are structured to integrate seamlessly with the AI Agent and MCP Server.  
- Edge cases include subworkflow failures, model errors, and credential issues.

---

#### 2.7 Model & Tool Configuration

**Overview:**  
Defines the AI models and tools used for natural language understanding, SQL generation, and MCP communication.

**Nodes Involved:**  
- Anthropic Chat Model  
- MCP Client  
- Think

**Node Details:**

- **Anthropic Chat Model**  
  - Used as the primary conversational model (Claude 3.5 Haiku).  
  - Optimized for MCP and multi-step reasoning.

- **MCP Client**  
  - Facilitates MCP protocol communication between AI Agent and MCP Server.

- **Think**  
  - Tool that decomposes complex queries into manageable subtasks for efficient processing.

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                      |
|--------------------------|----------------------------------|----------------------------------------------|------------------------------|-----------------------------|---------------------------------|
| PostgreSQL MCP Server    | MCP Trigger                      | Receives user queries via MCP endpoint       | External HTTP requests        | AI Agent, Schema Tools       |                                 |
| When chat message received| Chat Trigger                    | Alternative chat-based input trigger          | External chat messages        | AI Agent                    |                                 |
| When Executed by Another Workflow | Execute Workflow Trigger  | Allows external workflow execution            | External workflow calls       | Operation                   |                                 |
| AI Agent                 | LangChain Agent                 | Core reasoning and multi-step query planning | MCP Server, Chat Trigger      | Operation, Think            |                                 |
| Simple Memory            | Memory Buffer Window            | Maintains conversation context                | AI Agent                     | AI Agent                    |                                 |
| Anthropic Chat Model     | Language Model (Claude 3.5)     | Provides NLP capabilities                      | AI Agent                     | AI Agent                    |                                 |
| MCP Client               | MCP Client Tool                 | Facilitates MCP communication                  | AI Agent                     | MCP Server                  |                                 |
| Think                    | Tool Think                     | Breaks down complex queries                     | AI Agent                     | AI Agent                    |                                 |
| ListTables               | Postgres Tool                  | Retrieves list of tables                        | MCP Server                   | MCP Server, AI Agent        |                                 |
| GetTableSchema           | Postgres Tool                  | Retrieves table schema                          | MCP Server                   | MCP Server, AI Agent        |                                 |
| get table details        | Tool Workflow (LangChain)       | Processes schema info into structured metadata | MCP Server                   | AI Agent                    |                                 |
| ReadTableRows            | Tool Workflow (LangChain)       | Reads table rows based on AI queries           | MCP Server                   | AI Agent                    |                                 |
| CreateTableRecords       | Tool Workflow (LangChain)       | Batch inserts records securely                  | MCP Server                   | AI Agent                    |                                 |
| UpdateTableRecords       | Tool Workflow (LangChain)       | Batch updates records securely                   | MCP Server                   | AI Agent                    |                                 |
| Operation                | Switch                        | Routes CRUD operations based on AI instructions | When Executed by Another Workflow | ReadTableRecord, CreateTableRecord, UpdateTableRecord |                                 |
| ReadTableRecord          | Postgres                      | Executes SELECT queries                          | Operation                    |                             |                                 |
| CreateTableRecord        | Postgres                      | Executes INSERT queries                          | Operation                    |                             |                                 |
| UpdateTableRecord        | Postgres                      | Executes UPDATE queries                          | Operation                    |                             |                                 |
| Sticky Note              | Sticky Note                   | Comments / documentation                         |                              |                             |                                 |
| Sticky Note1             | Sticky Note                   | Comments / documentation                         |                              |                             |                                 |
| Sticky Note2             | Sticky Note                   | Comments / documentation                         |                              |                             |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL MCP Server Trigger Node**  
   - Type: MCP Trigger  
   - Configure webhook (default `/mcp/...` endpoint)  
   - No parameters needed  
   - Purpose: Receive user queries via MCP protocol.

2. **Create Chat Trigger Node**  
   - Type: Chat Trigger  
   - Configure webhook for chat messages  
   - Purpose: Alternative entry point for chat-based queries.

3. **Create Execute Workflow Trigger Node**  
   - Type: Execute Workflow Trigger  
   - No parameters  
   - Purpose: Allow external workflows to invoke CRUD operations.

4. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Connect Anthropic Chat Model as language model  
   - Connect Simple Memory node for conversation context  
   - Connect MCP Client tool for MCP communication  
   - Purpose: Process user input, plan multi-step queries.

5. **Create Simple Memory Node**  
   - Type: Memory Buffer Window  
   - Default parameters  
   - Connect to AI Agent’s memory input/output.

6. **Create Anthropic Chat Model Node**  
   - Type: LangChain LM Chat Anthropic  
   - Configure with Anthropic API credentials (Claude 3.5 Haiku)  
   - Connect to AI Agent’s language model input.

7. **Create MCP Client Node**  
   - Type: MCP Client Tool  
   - Default parameters  
   - Connect to AI Agent’s tool input and MCP Server output.

8. **Create Think Node**  
   - Type: Tool Think  
   - Default parameters  
   - Connect AI Agent output to Think input, Think output back to AI Agent.

9. **Create ListTables Node**  
   - Type: Postgres Tool  
   - Configure with PostgreSQL credentials  
   - Connect MCP Server output to ListTables input.

10. **Create GetTableSchema Node**  
    - Type: Postgres Tool  
    - Configure with PostgreSQL credentials  
    - Connect MCP Server output to GetTableSchema input.

11. **Create Tool Workflow Nodes for DB Operations**  
    - Create `get table details` (LangChain toolWorkflow)  
    - Create `ReadTableRows` (LangChain toolWorkflow)  
    - Create `CreateTableRecords` (LangChain toolWorkflow)  
    - Create `UpdateTableRecords` (LangChain toolWorkflow)  
    - Configure each with DeepSeek model credentials  
    - Connect MCP Server output to each toolWorkflow input.

12. **Create Operation Switch Node**  
    - Type: Switch  
    - Configure to switch on operation type (read, create, update)  
    - Connect "When Executed by Another Workflow" output to Operation input.

13. **Create Postgres Nodes for CRUD**  
    - ReadTableRecord (Postgres) for SELECT  
    - CreateTableRecord (Postgres) for INSERT  
    - UpdateTableRecord (Postgres) for UPDATE  
    - Configure all with PostgreSQL credentials  
    - Connect Operation outputs accordingly.

14. **Connect AI Agent output to Operation node**  
    - AI Agent’s planned operations trigger CRUD dispatcher.

15. **Upload and link the `checkdatabase` subworkflow**  
    - Import `checkdatabase.json` into n8n  
    - Configure PostgreSQL credentials inside subworkflow  
    - Ensure it is invoked by relevant toolWorkflow nodes for SQL validation and formatting.

16. **Set Credentials**  
    - PostgreSQL credentials with appropriate permissions (read, write)  
    - Anthropic API credentials for Claude 3.5 Haiku  
    - DeepSeek API credentials for SQL generation subworkflows

17. **Test the workflow**  
    - Use `/mcp/...` endpoint or chat trigger to send multi-KPI natural language queries  
    - Confirm AI Agent breaks down queries, generates SQL, executes safely, and returns consolidated reports.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Estimated cost per full multi-request run is approximately $0.01, optimized for speed and affordability.       | Cost efficiency note from workflow description.                                                  |
| This workflow avoids raw SQL execution for enhanced security by using parameterized queries and schema checks. | Security best practices.                                                                         |
| Recommended models: Claude 3.5 Haiku (Anthropic) for MCP Agent, DeepSeek for SQL generation subworkflow.       | Model selection recommendations.                                                                |
| Supports multi-KPI questions in a single message, returning structured, human-readable reports.                 | Key feature differentiating from original Conversing with Data template.                         |
| MCP Server Trigger URL (`/mcp/...`) is the main integration endpoint for frontend or chatbot connections.      | Integration instruction.                                                                         |
| For a lighter alternative, see the original Conversing with Data workflow: [Conversing with Data](https://n8n.io/workflows/3497-conversing-with-data-transforming-text-into-sql-queries-and-visual-curves) | Alternative workflow link.                                                                       |
| Additional workflows by the same author include Customer Feedback Analysis with AI, QuickChart & HTML Report Generator. | Related workflows link: [Customer Feedback Analysis](https://n8n.io/workflows/3642-customer-feedback-analysis-with-ai-quickchart-and-html-report-generator) |

---

This documentation provides a comprehensive understanding of the PostgreSQL Conversational Agent workflow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot it effectively.