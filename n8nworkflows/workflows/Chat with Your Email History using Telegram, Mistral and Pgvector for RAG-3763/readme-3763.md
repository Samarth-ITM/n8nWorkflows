Chat with Your Email History using Telegram, Mistral and Pgvector for RAG

https://n8nworkflows.xyz/workflows/chat-with-your-email-history-using-telegram--mistral-and-pgvector-for-rag-3763


# Chat with Your Email History using Telegram, Mistral and Pgvector for RAG

### 1. Workflow Overview

This workflow enables conversational querying of your entire email history using Telegram as the chat interface, combined with advanced retrieval-augmented generation (RAG) techniques powered by vector similarity search and structured SQL queries. It is designed for users who want to ask natural language questions about their past emails (e.g., "What hotel did I stay at last summer?") and receive precise, context-aware answers.

The workflow is architected around these logical blocks:

- **1.1 Input Reception:** Captures user queries either from Telegram messages or n8n’s internal chat interface.
- **1.2 Session Preparation:** Normalizes and prepares the incoming chat input for processing.
- **1.3 AI Agent Processing:** Uses a LangChain AI agent that orchestrates calls to two tools: a vector similarity search over email embeddings stored in a Postgres/pgvector database, and a structured SQL query workflow for factual data retrieval.
- **1.4 Embeddings Generation & Vector Search:** Generates vector embeddings for queries and retrieves semantically relevant emails.
- **1.5 Structured SQL Query Tool:** Invokes a sub-workflow that translates natural language queries into SQL to query structured email metadata.
- **1.6 Response Formatting & Delivery:** Processes the AI agent’s output, splits long responses into manageable chunks, escapes Markdown special characters, and sends the answer back to the user on Telegram in batches.

This design leverages local open-source models and infrastructure (Ollama for embeddings, Mistral for language modeling, pgvector for vector search) to ensure privacy and control over data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming messages either from Telegram or n8n’s chat interface to trigger the workflow.
- **Nodes Involved:**  
  - Telegram Trigger  
  - When chat message received

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Listens for new Telegram messages from a specific chat ID (configured to chat ID 6865163996)  
    - Configuration: Listens to "message" updates only; uses Telegram API credentials for bot authentication  
    - Inputs: External Telegram webhook  
    - Outputs: Emits Telegram message data  
    - Edge cases: Telegram API downtime, invalid chat ID, webhook misconfiguration

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Listens for chat messages within n8n’s internal chat interface  
    - Configuration: Default options, webhook enabled  
    - Inputs: External webhook from n8n chat  
    - Outputs: Emits chat message data  
    - Edge cases: Webhook failures, malformed chat messages

#### 2.2 Session Preparation

- **Overview:** Normalizes incoming message data into a consistent session format for downstream processing.
- **Nodes Involved:**  
  - Generate session id

- **Node Details:**

  - **Generate session id**  
    - Type: Set node  
    - Role: Extracts and formats chat input text, reply-to message ID, and message ID into a JSON object  
    - Configuration: Uses expressions to safely quote and extract text from Telegram or chat inputs  
    - Key expressions:  
      - `chatInput`: Extracts quoted text from Telegram message or chat input  
      - `reply_to`: Extracts reply-to message ID if present  
      - `message_id`: Uses sessionId or Telegram message ID  
    - Inputs: Telegram Trigger or Chat Trigger output  
    - Outputs: Normalized session data for AI Agent  
    - Edge cases: Missing fields, unexpected message formats

#### 2.3 AI Agent Processing

- **Overview:** Core logic that uses a LangChain AI Agent to process the user query by invoking vector and SQL search tools, guided by a detailed system prompt.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Postgres PGVector Store  
  - Call the SQL composer Workflow  
  - Simple Memory

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Orchestrates the query answering process by calling vector and SQL search tools  
    - Configuration:  
      - Input text from `chatInput`  
      - System prompt instructs the agent on how to use tools:  
        - Use vector search tool `emails_vector_search` for semantic email text search  
        - Use SQL tool `email_sql_search` for structured queries, linking results by email ID  
        - Detailed instructions on time handling and fallback logic  
      - Prompt disables human override of system message  
      - Strips markdown from answers  
    - Inputs: Normalized session data  
    - Outputs: AI-generated response text  
    - Edge cases: Model API errors, tool invocation failures, incomplete data, prompt misinterpretation

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Provides the language model backend for the AI Agent  
    - Configuration: Uses Mistral-small 3.1 latest model via Ollama API credentials  
    - Inputs: AI Agent language model requests  
    - Outputs: Model-generated text completions  
    - Edge cases: API rate limits, model unavailability

  - **Postgres PGVector Store**  
    - Type: LangChain Vector Store PGVector node  
    - Role: Performs vector similarity search on email embeddings stored in Postgres with pgvector extension  
    - Configuration:  
      - Mode: retrieve-as-tool (exposed as a tool to AI Agent)  
      - Top K: 100 results  
      - Table: `emails_embeddings`  
      - Tool description includes instructions for time-specific queries and metadata linking  
      - Uses Postgres credentials  
    - Inputs: Embeddings from Ollama or queries from AI Agent  
    - Outputs: Vector search results with metadata  
    - Edge cases: DB connection issues, empty results, query timeouts

  - **Call the SQL composer Workflow**  
    - Type: LangChain Tool Workflow node  
    - Role: Invokes a separate workflow to translate natural language queries into SQL and execute them on structured email data  
    - Configuration:  
      - Workflow ID points to "email_sql_search" template  
      - Input: `natural_language_query` passed from AI Agent  
      - Tool description guides usage for structured queries  
    - Inputs: AI Agent tool calls  
    - Outputs: Structured query results  
    - Edge cases: Sub-workflow errors, SQL injection risks, malformed queries

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window node  
    - Role: Maintains conversational context using a session key based on message or reply IDs  
    - Configuration: Session key derived from `reply_to` or `message_id`  
    - Inputs: AI Agent context  
    - Outputs: Contextual memory for agent  
    - Edge cases: Memory overflow, session key collisions

#### 2.4 Embeddings Generation & Vector Search

- **Overview:** Generates vector embeddings for query text using Ollama’s `nomic-embed-text` model and feeds them into the Postgres PGVector Store for similarity search.
- **Nodes Involved:**  
  - Embeddings Ollama  
  - Postgres PGVector Store (already described above)

- **Node Details:**

  - **Embeddings Ollama**  
    - Type: LangChain Embeddings Ollama node  
    - Role: Converts text input into vector embeddings using the `nomic-embed-text:latest` model  
    - Configuration: Uses Ollama API credentials  
    - Inputs: Text from AI Agent or query  
    - Outputs: Vector embeddings  
    - Edge cases: API errors, embedding model unavailability

#### 2.5 Response Formatting & Delivery

- **Overview:** Formats the AI agent’s output, splits long text into chunks, escapes Markdown special characters, and sends the response back to Telegram in batches.
- **Nodes Involved:**  
  - Came from Telegram? (If node)  
  - Split text into chunks (Code node)  
  - Loop Over Items (Split in Batches)  
  - Escape Markdown (Code node)  
  - Respond on Telegram in batches  
  - Beautify chat response  
  - No Operation, do nothing

- **Node Details:**

  - **Came from Telegram?**  
    - Type: If node  
    - Role: Checks if the workflow was triggered by Telegram (boolean check on Telegram Trigger execution)  
    - Outputs:  
      - True: proceeds to split text into chunks for Telegram response  
      - False: proceeds to beautify chat response for n8n chat interface  
    - Edge cases: Incorrect trigger detection

  - **Split text into chunks**  
    - Type: Code node (JavaScript)  
    - Role: Splits long response text into chunks of max 500 characters, breaking at spaces to avoid mid-word splits  
    - Inputs: AI Agent output text  
    - Outputs: Array of text chunks for batch sending  
    - Edge cases: Very long texts, no spaces in text

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Role: Processes each text chunk sequentially for sending  
    - Configuration: Default batch size (1)  
    - Inputs: Text chunks from previous node  
    - Outputs: Single chunk per iteration  
    - Edge cases: Large number of chunks causing delays

  - **Escape Markdown**  
    - Type: Code node (JavaScript)  
    - Role: Escapes Markdown special characters in text to prevent formatting issues in Telegram messages  
    - Inputs: Text chunk  
    - Outputs: Escaped text chunk  
    - Edge cases: Unexpected characters, incomplete escaping

  - **Respond on Telegram in batches**  
    - Type: Telegram node  
    - Role: Sends each escaped text chunk as a Telegram message reply to the original user message  
    - Configuration:  
      - Chat ID from Telegram Trigger message sender  
      - Reply to original message ID  
      - MarkdownV2 parse mode enabled  
      - Notifications disabled, web page preview disabled  
    - Inputs: Escaped text chunks  
    - Outputs: Telegram message sent confirmation  
    - Edge cases: Telegram API rate limits, message size limits

  - **Beautify chat response**  
    - Type: Set node  
    - Role: Prepares output for n8n chat interface by assigning AI output to a field named `output`  
    - Inputs: AI Agent output  
    - Outputs: Formatted chat response  
    - Edge cases: Empty or malformed AI output

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Placeholder for batch processing path that does nothing (used for non-Telegram triggers)  
    - Inputs: Loop Over Items output  
    - Outputs: None  
    - Edge cases: None

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                  | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                              |
|----------------------------|--------------------------------------|-------------------------------------------------|------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                     | Receives Telegram messages                        |                              | Generate session id                |                                                                                                         |
| When chat message received  | LangChain Chat Trigger               | Receives n8n chat messages                        |                              | Generate session id                |                                                                                                         |
| Generate session id         | Set                                 | Normalizes chat input and session IDs            | Telegram Trigger, When chat message received | AI Agent                         |                                                                                                         |
| AI Agent                   | LangChain Agent                     | Core AI logic orchestrating vector and SQL tools | Generate session id           | Came from Telegram?, Simple Memory |                                                                                                         |
| OpenAI Chat Model           | LangChain LM Chat OpenAI             | Provides language model backend (Mistral)        | AI Agent                     | AI Agent                          |                                                                                                         |
| Postgres PGVector Store     | LangChain Vector Store PGVector      | Vector similarity search tool                      | Embeddings Ollama, AI Agent  | AI Agent                         |                                                                                                         |
| Call the SQL composer Workflow | LangChain Tool Workflow             | Executes structured SQL queries                    | AI Agent                     | AI Agent                         | "IMPORTANT\nFor this step to work, you must download my other template *Translate questions about e-mails into SQL queries and run them*." |
| Simple Memory               | LangChain Memory Buffer Window       | Maintains conversational context                   | AI Agent                     | AI Agent                         |                                                                                                         |
| Embeddings Ollama           | LangChain Embeddings Ollama          | Generates vector embeddings                         | AI Agent                     | Postgres PGVector Store          |                                                                                                         |
| Came from Telegram?         | If                                  | Checks if trigger was Telegram                      | AI Agent                     | Split text into chunks, Beautify chat response |                                                                                                         |
| Split text into chunks      | Code                                | Splits long text into chunks                        | Came from Telegram?          | Loop Over Items                  |                                                                                                         |
| Loop Over Items             | SplitInBatches                      | Processes text chunks sequentially                  | Split text into chunks       | Escape Markdown, No Operation     |                                                                                                         |
| Escape Markdown             | Code                                | Escapes Markdown special characters                 | Loop Over Items              | Respond on Telegram in batches    |                                                                                                         |
| Respond on Telegram in batches | Telegram                          | Sends response chunks back to Telegram              | Escape Markdown              | Loop Over Items                  |                                                                                                         |
| Beautify chat response      | Set                                 | Formats response for n8n chat interface             | Came from Telegram?          |                                | "Response\nThis section takes care of formatting the answer and either responding over Telegram, or in n8n's chat." |
| No Operation, do nothing    | NoOp                                | Placeholder for batch processing path               | Loop Over Items              |                                |                                                                                                         |
| Sticky Note                 | Sticky Note                         | Informational note                                  |                              |                                | "Chat around!\nYou can use this workflow both as a Telegram bot, or by chatting with it in n8n's interface." |
| Sticky Note1                | Sticky Note                         | Informational note                                  |                              |                                | "🤖 \nThis AI Agent has the mission to query both **structured** and **vectorized** databases containing all your e-mail communications.\n\nAdjust the *SQL composer Workflow* to point at a copy of my *Translate questions about e-mails into SQL queries and run them* template." |
| Sticky Note2                | Sticky Note                         | Informational note                                  |                              |                                | "IMPORTANT\nFor this step to work, you must download my other template *Translate questions about e-mails into SQL queries and run them*." |
| Sticky Note3                | Sticky Note                         | Informational note                                  |                              |                                | "Response\nThis section takes care of formatting the answer and either responding over Telegram, or in n8n's chat." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only  
   - Set `chatIds` to your Telegram chat ID (e.g., 6865163996)  
   - Authenticate with Telegram API credentials for your bot

2. **Create LangChain Chat Trigger node**  
   - Type: LangChain Chat Trigger  
   - Use default webhook settings to receive chat messages inside n8n

3. **Create Set node "Generate session id"**  
   - Extract and normalize input text:  
     - `chatInput`: Use expression to quote Telegram message text or chat input text  
     - `reply_to`: Extract reply-to message ID if present  
     - `message_id`: Use sessionId or Telegram message ID  
   - Connect outputs of Telegram Trigger and Chat Trigger to this node

4. **Create LangChain Agent node "AI Agent"**  
   - Input: `chatInput` from "Generate session id"  
   - Configure system prompt with detailed instructions on using two tools:  
     - Vector search tool named `emails_vector_search`  
     - SQL search tool named `email_sql_search`  
   - Disable human override of system prompt  
   - Strip markdown from answers  
   - Connect output of "Generate session id" to this node

5. **Create LangChain OpenAI Chat Model node**  
   - Model: `mistral-small3.1:latest` via Ollama API  
   - Connect as language model backend to "AI Agent"

6. **Create LangChain Embeddings Ollama node**  
   - Model: `nomic-embed-text:latest`  
   - Connect as embeddings provider to "Postgres PGVector Store"

7. **Create LangChain Vector Store PGVector node "Postgres PGVector Store"**  
   - Mode: `retrieve-as-tool`  
   - Top K: 100  
   - Table: `emails_embeddings`  
   - Tool name: `emails_vector_search`  
   - Tool description: Include instructions for time-specific queries and metadata linking  
   - Connect embeddings input from "Embeddings Ollama"  
   - Connect tool output to "AI Agent" as a tool

8. **Create LangChain Tool Workflow node "Call the SQL composer Workflow"**  
   - Set to call your sub-workflow that translates natural language to SQL and executes it on structured email data  
   - Workflow ID: your "email_sql_search" workflow  
   - Input: `natural_language_query` from AI Agent tool calls  
   - Tool description: Explain usage for structured email queries  
   - Connect tool output to "AI Agent"

9. **Create LangChain Memory Buffer Window node "Simple Memory"**  
   - Session key: Use `reply_to` or `message_id` from session data  
   - Connect memory input/output to "AI Agent"

10. **Create If node "Came from Telegram?"**  
    - Condition: Check if Telegram Trigger was executed (boolean true)  
    - Connect AI Agent output to this node

11. **Create Code node "Split text into chunks"**  
    - JavaScript code to split text into max 500-character chunks at spaces  
    - Connect True output of "Came from Telegram?" to this node

12. **Create SplitInBatches node "Loop Over Items"**  
    - Default batch size (1)  
    - Connect output of "Split text into chunks" to this node

13. **Create Code node "Escape Markdown"**  
    - JavaScript code to escape Markdown special characters for Telegram MarkdownV2  
    - Connect one output of "Loop Over Items" to this node

14. **Create Telegram node "Respond on Telegram in batches"**  
    - Text: Use escaped text from previous node  
    - Chat ID: Extract from Telegram Trigger message sender ID  
    - Reply to message ID: Original Telegram message ID  
    - Parse mode: MarkdownV2  
    - Disable notifications and web page preview  
    - Connect output of "Escape Markdown" to this node

15. **Create Set node "Beautify chat response"**  
    - Assign AI Agent output to field `output` for n8n chat interface  
    - Connect False output of "Came from Telegram?" to this node

16. **Create No Operation node "No Operation, do nothing"**  
    - Connect second output of "Loop Over Items" to this node (used as placeholder)

17. **Connect all nodes according to the logical flow:**  
    - Telegram Trigger and Chat Trigger → Generate session id → AI Agent  
    - AI Agent → Came from Telegram?  
    - Came from Telegram? True → Split text into chunks → Loop Over Items → Escape Markdown → Respond on Telegram in batches  
    - Came from Telegram? False → Beautify chat response  
    - Loop Over Items second output → No Operation

18. **Configure all credentials:**  
    - Telegram API credentials for Telegram nodes  
    - Ollama API credentials for embeddings and language model nodes  
    - Postgres credentials for PGVector node  
    - Ensure sub-workflow "email_sql_search" is imported and configured with access to your structured email database

19. **Activate the workflow** and test by sending messages via Telegram or n8n chat interface.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is designed to be 100% local using open-source models and infrastructure: Ollama for LLM and embeddings, pgvector extension on Postgres for vector search.                                                                                                                                                                                                                                                                                                                                  | Workflow description                                                                                             |
| Vector similarity search enables semantic querying of email text, while structured SQL queries provide factual metadata retrieval. Both are linked by email message IDs.                                                                                                                                                                                                                                                                                                                                  | Workflow description                                                                                             |
| For best results, download and use the companion template *Translate questions about e-mails into SQL queries and run them* to enable structured SQL querying.                                                                                                                                                                                                                                                                                                                                           | Sticky Note2 and Call the SQL composer Workflow node description                                                |
| The AI Agent system prompt includes detailed instructions on time handling (e.g., interpreting "next week", "upcoming") and fallback logic between vector and SQL tools.                                                                                                                                                                                                                                                                                                                                | AI Agent node configuration                                                                                      |
| Telegram responses are split into chunks of max 500 characters and escaped for MarkdownV2 to avoid formatting errors.                                                                                                                                                                                                                                                                                                                                                                                    | Split text into chunks, Escape Markdown, and Respond on Telegram nodes                                           |
| You can interact with this workflow either via Telegram bot or directly in n8n’s chat interface.                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note                                                                                                      |
| For more information on vector embeddings and similarity search, see https://www.pinecone.io/learn/vector-embeddings/                                                                                                                                                                                                                                                                                                                                                                                    | Workflow description                                                                                             |
| The structured email schema includes fields like date, thread_id, email_from, email_to, email_subject, attachments, email_id, and email_text.                                                                                                                                                                                                                                                                                                                                                              | AI Agent system prompt                                                                                            |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Chat with Your Email History using Telegram, Mistral and Pgvector for RAG" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this solution.