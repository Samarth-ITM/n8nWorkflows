MCP Supabase Server for AI Agent with RAG & Multi-Tenant CRUD

https://n8nworkflows.xyz/workflows/mcp-supabase-server-for-ai-agent-with-rag---multi-tenant-crud-3675


# MCP Supabase Server for AI Agent with RAG & Multi-Tenant CRUD

### 1. Workflow Overview

This workflow, titled **MCP Supabase Server for AI Agent with RAG & Multi-Tenant CRUD**, is designed to implement a stateful AI agent leveraging Supabase as a backend and Retrieval-Augmented Generation (RAG) for context-aware AI responses. It supports multi-tenant data isolation, enabling secure and dynamic CRUD operations on multiple agent-related data tables. The workflow is suitable for AI-driven applications such as customer support chatbots, automated task management, and knowledge repositories.

The workflow logic is organized into the following functional blocks:

- **1.1 Input Reception and AI Agent Trigger**  
  Handles incoming requests via a custom MCP trigger node that acts as the AI agent’s entry point.

- **1.2 RAG Processing and Embeddings Generation**  
  Uses OpenAI embeddings and Supabase vector search to retrieve relevant context for AI responses.

- **1.3 Agent Messages CRUD Operations**  
  Manages create, read, update, delete, and list operations on the `agent_messages` table.

- **1.4 Agent Tasks CRUD Operations**  
  Manages CRUD operations on the `agent_tasks` table.

- **1.5 Agent Status CRUD Operations**  
  Manages CRUD operations on the `agent_status` table.

- **1.6 Agent Knowledge CRUD Operations**  
  Manages CRUD operations on the `agent_knowledge` table.

- **1.7 Auxiliary and Documentation Nodes**  
  Sticky notes provide logical separation and documentation for each data domain.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and AI Agent Trigger

- **Overview:**  
  This block receives incoming requests to the AI agent via a custom MCP trigger node. It serves as the main entry point for all AI-related operations and routes commands to the appropriate CRUD or RAG nodes.

- **Nodes Involved:**  
  - MCP_SUPABASE

- **Node Details:**  
  - **MCP_SUPABASE**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Custom trigger node for AI agent requests, supports multi-tenant routing via webhook path.  
    - Configuration: Webhook path set to a unique identifier (`affff59c-9c5c-4a07-b531-616c1d631601`), which can be replaced with a dynamic path like `/mcp/tool/supabase/:userId` for multi-tenancy.  
    - Inputs: External HTTP requests via webhook.  
    - Outputs: Routes to all CRUD and RAG nodes via `ai_tool` connections.  
    - Edge Cases: Potential webhook path conflicts, authentication or authorization failures if RLS policies are misconfigured.  
    - Version: Compatible with n8n 1.88.0+.

#### 1.2 RAG Processing and Embeddings Generation

- **Overview:**  
  This block generates embeddings for input text using OpenAI and performs vector similarity search in Supabase to retrieve relevant context for AI responses, enabling Retrieval-Augmented Generation.

- **Nodes Involved:**  
  - Embeddings OpenAI  
  - RAG

- **Node Details:**  
  - **Embeddings OpenAI**  
    - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
    - Role: Generates text embeddings using OpenAI’s `text-embedding-ada-002` model.  
    - Configuration: Model set to `text-embedding-ada-002`, no additional options.  
    - Inputs: Text data from MCP_SUPABASE node.  
    - Outputs: Embeddings passed to RAG node.  
    - Credentials: Requires OpenAI API key.  
    - Edge Cases: API rate limits, network timeouts, invalid input text.  
    - Version: 1.2.

  - **RAG**  
    - Type: `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
    - Role: Performs vector similarity search in Supabase vector store to retrieve top 5 relevant documents.  
    - Configuration:  
      - Mode: `retrieve-as-tool` (used as a tool in AI chain)  
      - topK: 5 (returns top 5 matches)  
      - Table: `documents` (Supabase vector store table)  
      - Tool Name: `ITERACOES`  
      - Tool Description: Remembers interactions and consults system instructions, also stores learned information.  
    - Inputs: Embeddings from OpenAI node.  
    - Outputs: Retrieved documents for AI context.  
    - Credentials: Requires Supabase API key.  
    - Edge Cases: Empty or missing vector store, API errors, malformed embeddings.  
    - Version: 1.1.

#### 1.3 Agent Messages CRUD Operations

- **Overview:**  
  This block handles all CRUD operations on the `agent_messages` table, which stores conversation messages for the AI agent.

- **Nodes Involved:**  
  - CREATE_ROW_AGENT_MESSAGE  
  - GET_ROW_AGENT_MESSAGE  
  - UPDATE_ROW_AGENT_MESSAGE  
  - DELETE_ROW_AGENT_MESSAGE  
  - GET_MANY_ROW_AGENT_MESSAGE  
  - Sticky Note (AGENT_MESSAGE)

- **Node Details:**  
  - **CREATE_ROW_AGENT_MESSAGE**  
    - Type: `supabaseTool`  
    - Role: Inserts new message rows into `agent_messages`.  
    - Configuration: Table set to `agent_messages`, operation defaults to create.  
    - Inputs: From MCP_SUPABASE node.  
    - Outputs: Confirmation of row creation.  
    - Credentials: Supabase API key.  
    - Edge Cases: Validation errors, RLS permission denials.

  - **GET_ROW_AGENT_MESSAGE**  
    - Type: `supabaseTool`  
    - Role: Retrieves a single message row by primary key or filter.  
    - Configuration: Table `agent_messages`, operation `get`.  
    - Inputs: From MCP_SUPABASE node.  
    - Outputs: Single row data.  
    - Edge Cases: Row not found, permission errors.

  - **UPDATE_ROW_AGENT_MESSAGE**  
    - Type: `supabaseTool`  
    - Role: Updates existing message rows.  
    - Configuration: Table `agent_messages`, operation `update`.  
    - Inputs: From MCP_SUPABASE node.  
    - Outputs: Updated row confirmation.  
    - Edge Cases: Row not found, concurrency conflicts.

  - **DELETE_ROW_AGENT_MESSAGE**  
    - Type: `supabaseTool`  
    - Role: Deletes message rows.  
    - Configuration: Table `agent_messages`, operation `delete`.  
    - Inputs: From MCP_SUPABASE node.  
    - Outputs: Deletion confirmation.  
    - Edge Cases: Row not found, permission errors.

  - **GET_MANY_ROW_AGENT_MESSAGE**  
    - Type: `supabaseTool`  
    - Role: Retrieves multiple message rows with optional limit.  
    - Configuration: Table `agent_messages`, operation `getAll`, limit parameter dynamically set via AI override expression.  
    - Inputs: From MCP_SUPABASE node.  
    - Outputs: List of rows.  
    - Edge Cases: Large data sets causing timeouts.

  - **Sticky Note (AGENT_MESSAGE)**  
    - Provides a visual label grouping these nodes under the "AGENT_MESSAGE" domain.

#### 1.4 Agent Tasks CRUD Operations

- **Overview:**  
  Manages CRUD operations on the `agent_tasks` table, which tracks tasks related to the AI agent.

- **Nodes Involved:**  
  - CREATE_ROW_AGENT_TASKS  
  - GET_ROW_AGENT_TASKS  
  - UPDATE_ROW_AGENT_TASKS  
  - DELETE_ROW_AGENT_TASKS  
  - GET_MANY_ROW_AGENT_TASKS  
  - Sticky Note1 (AGENT_TASK)

- **Node Details:**  
  - Similar structure and roles as the Agent Messages CRUD nodes, but operating on the `agent_tasks` table.

#### 1.5 Agent Status CRUD Operations

- **Overview:**  
  Handles CRUD operations on the `agent_status` table, which stores status information for the AI agent.

- **Nodes Involved:**  
  - CREATE_ROW_AGENT_STATUS  
  - GET_ROW_AGENT_STATUS  
  - UPDATE_ROW_AGENT_STATUS  
  - DELETE_ROW_AGENT_STATUS  
  - GET_MANY_ROW_AGENT_STATUS  
  - Sticky Note2 (AGENT_STATUS)

- **Node Details:**  
  - Same pattern as previous CRUD blocks, targeting the `agent_status` table.

#### 1.6 Agent Knowledge CRUD Operations

- **Overview:**  
  Manages CRUD operations on the `agent_knowledge` table, which holds domain-specific knowledge for the AI agent.

- **Nodes Involved:**  
  - CREATE_ROW_AGENT_KNOWLEDGE  
  - GET_ROW_AGENT_KNOWLEDGE  
  - UPDATE_ROW_INSCRICOES_AGENT_KNOWLEDGE  
  - DELETE_ROW_INSCRICOES_CURSOS  
  - GET_MANY_ROW_AGENT_KNOWLEDGE  
  - Sticky Note3 (AGENT_KNOWLEDGE)

- **Node Details:**  
  - **CREATE_ROW_AGENT_KNOWLEDGE**: Inserts new knowledge entries.  
  - **GET_ROW_AGENT_KNOWLEDGE**: Retrieves single knowledge entries.  
  - **UPDATE_ROW_INSCRICOES_AGENT_KNOWLEDGE**: Updates knowledge entries.  
  - **DELETE_ROW_INSCRICOES_CURSOS**: Deletes knowledge entries (named differently but operates on `agent_knowledge` table).  
  - **GET_MANY_ROW_AGENT_KNOWLEDGE**: Retrieves multiple knowledge entries with limit.  
  - Edge Cases: Same as other CRUD nodes, plus potential naming confusion due to node names not fully aligned with table names.

#### 1.7 Auxiliary and Documentation Nodes

- **Overview:**  
  Sticky notes are used to visually separate and document the different data domains (`agent_messages`, `agent_tasks`, `agent_status`, `agent_knowledge`).

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  
  - Type: `stickyNote`  
  - Role: Provide clear visual grouping and documentation within the workflow editor.  
  - Content: Titles corresponding to each data domain.

---

### 3. Summary Table

| Node Name                      | Node Type                                    | Functional Role                     | Input Node(s)    | Output Node(s)   | Sticky Note                      |
|-------------------------------|----------------------------------------------|-----------------------------------|------------------|------------------|---------------------------------|
| MCP_SUPABASE                  | @n8n/n8n-nodes-langchain.mcpTrigger          | AI agent entry trigger             | External webhook | All CRUD & RAG   |                                 |
| Embeddings OpenAI             | @n8n/n8n-nodes-langchain.embeddingsOpenAi    | Generate OpenAI text embeddings    | MCP_SUPABASE     | RAG              |                                 |
| RAG                          | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Retrieve context via vector search | Embeddings OpenAI| MCP_SUPABASE     |                                 |
| CREATE_ROW_AGENT_MESSAGE      | supabaseTool                                 | Create message row                 | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_MESSAGE                   |
| GET_ROW_AGENT_MESSAGE         | supabaseTool                                 | Get single message row             | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_MESSAGE                   |
| UPDATE_ROW_AGENT_MESSAGE      | supabaseTool                                 | Update message row                 | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_MESSAGE                   |
| DELETE_ROW_AGENT_MESSAGE      | supabaseTool                                 | Delete message row                 | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_MESSAGE                   |
| GET_MANY_ROW_AGENT_MESSAGE    | supabaseTool                                 | Get multiple message rows          | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_MESSAGE                   |
| CREATE_ROW_AGENT_TASKS        | supabaseTool                                 | Create task row                   | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_TASK                     |
| GET_ROW_AGENT_TASKS           | supabaseTool                                 | Get single task row               | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_TASK                     |
| UPDATE_ROW_AGENT_TASKS        | supabaseTool                                 | Update task row                   | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_TASK                     |
| DELETE_ROW_AGENT_TASKS        | supabaseTool                                 | Delete task row                   | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_TASK                     |
| GET_MANY_ROW_AGENT_TASKS      | supabaseTool                                 | Get multiple task rows            | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_TASK                     |
| CREATE_ROW_AGENT_STATUS       | supabaseTool                                 | Create status row                 | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_STATUS                   |
| GET_ROW_AGENT_STATUS          | supabaseTool                                 | Get single status row             | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_STATUS                   |
| UPDATE_ROW_AGENT_STATUS       | supabaseTool                                 | Update status row                 | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_STATUS                   |
| DELETE_ROW_AGENT_STATUS       | supabaseTool                                 | Delete status row                 | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_STATUS                   |
| GET_MANY_ROW_AGENT_STATUS     | supabaseTool                                 | Get multiple status rows          | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_STATUS                   |
| CREATE_ROW_AGENT_KNOWLEDGE    | supabaseTool                                 | Create knowledge row              | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_KNOWLEDGE               |
| GET_ROW_AGENT_KNOWLEDGE       | supabaseTool                                 | Get single knowledge row          | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_KNOWLEDGE               |
| UPDATE_ROW_INSCRICOES_AGENT_KNOWLEDGE | supabaseTool                         | Update knowledge row              | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_KNOWLEDGE               |
| DELETE_ROW_INSCRICOES_CURSOS  | supabaseTool                                 | Delete knowledge row              | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_KNOWLEDGE               |
| GET_MANY_ROW_AGENT_KNOWLEDGE  | supabaseTool                                 | Get multiple knowledge rows       | MCP_SUPABASE     | MCP_SUPABASE     | AGENT_KNOWLEDGE               |
| Sticky Note                   | stickyNote                                   | Visual label for AGENT_MESSAGE    | None             | None             | AGENT_MESSAGE                 |
| Sticky Note1                  | stickyNote                                   | Visual label for AGENT_TASK       | None             | None             | AGENT_TASK                    |
| Sticky Note2                  | stickyNote                                   | Visual label for AGENT_STATUS     | None             | None             | AGENT_STATUS                  |
| Sticky Note3                  | stickyNote                                   | Visual label for AGENT_KNOWLEDGE  | None             | None             | AGENT_KNOWLEDGE               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP_SUPABASE Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set webhook path to a unique identifier or dynamic path (e.g., `/mcp/tool/supabase/:userId`) for multi-tenancy.  
   - No credentials needed here.  
   - Position: Entry point.

2. **Create Embeddings OpenAI Node**  
   - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
   - Configure model: `text-embedding-ada-002`  
   - Connect input from MCP_SUPABASE node’s relevant output.  
   - Attach OpenAI API credentials.

3. **Create RAG Node (Supabase Vector Store)**  
   - Type: `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
   - Set mode: `retrieve-as-tool`  
   - Set topK: 5  
   - Table name: `documents` (or dynamic if multi-tenant)  
   - Tool name: `ITERACOES`  
   - Tool description: "Remembers interactions and consults system instructions, also stores learned information."  
   - Connect input from Embeddings OpenAI node output.  
   - Attach Supabase API credentials.

4. **Create Agent Messages CRUD Nodes**  
   - Create five `supabaseTool` nodes for `agent_messages` table:  
     - CREATE_ROW_AGENT_MESSAGE (default create operation)  
     - GET_ROW_AGENT_MESSAGE (operation: get)  
     - UPDATE_ROW_AGENT_MESSAGE (operation: update)  
     - DELETE_ROW_AGENT_MESSAGE (operation: delete)  
     - GET_MANY_ROW_AGENT_MESSAGE (operation: getAll, with limit parameter)  
   - Connect all these nodes’ `ai_tool` input to MCP_SUPABASE node.  
   - Attach Supabase API credentials.

5. **Create Agent Tasks CRUD Nodes**  
   - Repeat step 4 for `agent_tasks` table with nodes:  
     - CREATE_ROW_AGENT_TASKS  
     - GET_ROW_AGENT_TASKS  
     - UPDATE_ROW_AGENT_TASKS  
     - DELETE_ROW_AGENT_TASKS  
     - GET_MANY_ROW_AGENT_TASKS  
   - Connect inputs from MCP_SUPABASE node.  
   - Attach Supabase API credentials.

6. **Create Agent Status CRUD Nodes**  
   - Repeat step 4 for `agent_status` table with nodes:  
     - CREATE_ROW_AGENT_STATUS  
     - GET_ROW_AGENT_STATUS  
     - UPDATE_ROW_AGENT_STATUS  
     - DELETE_ROW_AGENT_STATUS  
     - GET_MANY_ROW_AGENT_STATUS  
   - Connect inputs from MCP_SUPABASE node.  
   - Attach Supabase API credentials.

7. **Create Agent Knowledge CRUD Nodes**  
   - Create nodes for `agent_knowledge` table:  
     - CREATE_ROW_AGENT_KNOWLEDGE (create)  
     - GET_ROW_AGENT_KNOWLEDGE (get)  
     - UPDATE_ROW_INSCRICOES_AGENT_KNOWLEDGE (update)  
     - DELETE_ROW_INSCRICOES_CURSOS (delete)  
     - GET_MANY_ROW_AGENT_KNOWLEDGE (getAll with limit)  
   - Connect inputs from MCP_SUPABASE node.  
   - Attach Supabase API credentials.

8. **Add Sticky Note Nodes for Documentation**  
   - Add four sticky notes with content:  
     - "## AGENT_MESSAGE" near message CRUD nodes  
     - "## AGENT_TASK" near task CRUD nodes  
     - "## AGENT_STATUS" near status CRUD nodes  
     - "## AGENT_KNOWLEDGE" near knowledge CRUD nodes

9. **Configure Credentials**  
   - Add and configure Supabase API credentials with appropriate API keys and permissions.  
   - Add and configure OpenAI API credentials.

10. **Set Multi-Tenancy (Optional)**  
    - Modify MCP_SUPABASE webhook path to include dynamic userId parameter.  
    - Use Set nodes (not included in this workflow) to dynamically generate table names per tenant, e.g., `agent_messages_{{userId}}`.  
    - Ensure Supabase RLS policies enforce row-level security per tenant.

11. **Activate Workflow and Test**  
    - Enable the workflow.  
    - Send test requests to the webhook URL.  
    - Monitor logs for errors and validate CRUD operations and RAG responses.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow version 1.0.0, compatible with n8n version 1.88.0+                                      | Versioning information from workflow metadata                                                  |
| Author: Koresolucoes                                                                             | Author credit                                                                                   |
| Licensed under MIT License                                                                       | License details                                                                                |
| Multi-Tenant setup instructions: Replace webhook path with `/mcp/tool/supabase/:userId`          | Enables per-user routing                                                                       |
| Use Set node to dynamically generate table names for multi-tenancy (e.g., `agent_messages_{{userId}}`) | Multi-tenant data isolation instructions                                                      |
| Key features include RAG integration, full CRUD, multi-tenant support, and secure RLS enforcement | Workflow capabilities summary                                                                  |
| Screenshots available in original workflow file for visual reference                             | Visual aids (not included here)                                                                |
| For OpenAI API key setup, see n8n documentation on OpenAI credentials                            | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.openai/                       |
| For Supabase API key setup, see n8n documentation on Supabase credentials                        | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.supabase/                     |

---

This document provides a comprehensive reference to understand, reproduce, and modify the MCP Supabase AI Agent workflow with RAG and multi-tenant CRUD capabilities.