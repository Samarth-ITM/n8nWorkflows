Build a local RAG chatbot with Ollama, Qwen, BGE-M3 and Postgres PGVector

https://n8nworkflows.xyz/workflows/build-a-local-rag-chatbot-with-ollama--qwen--bge-m3-and-postgres-pgvector-14782


# Build a local RAG chatbot with Ollama, Qwen, BGE-M3 and Postgres PGVector

# Workflow Analysis: Local RAG Chatbot (Ollama, Qwen, BGE-M3, Postgres)

### 1. Workflow Overview

This workflow implements a fully local Retrieval-Augmented Generation (RAG) chatbot. It is designed to run without cloud APIs, utilizing Ollama for local LLM hosting and PostgreSQL with the `pgvector` extension for document storage and retrieval. 

The core logic separates "small talk" from "knowledge retrieval" using a classification step. Instead of relying on an AI Agent's internal tool-calling (which can be unreliable on smaller models), this workflow uses deterministic n8n nodes to handle the retrieval loop, filtering, and data aggregation.

**Logical Blocks:**
- **1.1 Input Reception & Classification:** Receives user input via Webhook and determines if the input is a question requiring RAG or a general conversation.
- **1.2 Routing:** Directs the flow based on the classification (Question vs. Discussion).
- **1.3 Small Talk Path:** A conversational loop using a larger model and session memory.
- **1.4 RAG Retrieval Pipeline:** A loop that decomposes complex questions into sub-queries, retrieves chunks from PGVector, and filters them by relevance score.
- **1.5 Answer Generation:** Synthesizes the final response using retrieved data and cleanses the output of reasoning tags.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Classification
**Overview:** This block captures the user request and uses a lightweight model to plan the retrieval strategy.
- **Nodes Involved:** `Webhook`, `Understand Request`, `Ollama Chat Model (Classifier — Qwen2.5:7b)`, `JSON Formatter`.
- **Node Details:**
  - **Webhook:** Listens for POST requests containing `chatInput` and `session_id`.
  - **Understand Request (LLM Chain):** Acts as a query planner. It classifies the intent (1–5 taxonomy) and generates a JSON list of sub-queries.
  - **Ollama Chat Model (Classifier):** Uses `qwen2.5:7b` for fast, efficient classification.
  - **JSON Formatter (Code):** A JavaScript node that cleans the LLM string output and parses it into a valid JSON object (`is_question`, `queries`, etc.) to prevent workflow crashes.

#### 2.2 Routing
**Overview:** Determines the execution path based on whether the user is asking a factual question.
- **Nodes Involved:** `Switch`.
- **Node Details:**
  - **Switch:** Evaluates the `is_question` boolean. If `false` $\rightarrow$ Small Talk Path; if `true` $\rightarrow$ RAG Retrieval Pipeline.

#### 2.3 Small Talk Path
**Overview:** Handles casual interaction without searching the knowledge base.
- **Nodes Involved:** `Small Talk AI Agent`, `Ollama Chat Model (Small Talk — Qwen3:14b)`, `Postgres Chat Memory (Small Talk)`, `Remove Think Tags (Small Talk Path)`.
- **Node Details:**
  - **Small Talk AI Agent:** A conversational agent with a system prompt focusing on professional greetings and boundary management.
  - **Ollama Chat Model:** Uses `qwen3:14b` for higher-quality conversational nuances.
  - **Postgres Chat Memory:** Stores conversation history in the `chat_histories` table using the `session_id`.
  - **Remove Think Tags (Code):** Uses a Regex to strip `<think>...</think>` blocks typical of reasoning models.

#### 2.4 RAG Retrieval Pipeline
**Overview:** The "engine" of the RAG process. It iterates through generated sub-queries to gather comprehensive evidence.
- **Nodes Involved:** `Split Sub-Queries`, `Loop Over Sub-Queries`, `PGVector Store — Retrieve Chunks`, `Ollama Embeddings (BGE-M3)`, `Clean RAG output`, `Keep score over 0.4`, `Any chunk?`, `Say no chunk match`, `Aggregate Matching Chunks`, `Prepare loop output`, `Aggregate All Retrieval Results`.
- **Node Details:**
  - **Split Sub-Queries:** Turns the `queries` array into individual items for the loop.
  - **Loop Over Sub-Queries (Split in Batches):** Ensures each sub-query is processed sequentially.
  - **PGVector Store:** Retrieves chunks based on embeddings.
  - **Ollama Embeddings:** Uses the `bge-m3` model for high-accuracy vectorization.
  - **Clean RAG output (Set):** Extracts `pageContent` and maps metadata (title, file path) and the relevance score.
  - **Keep score over 0.4 (Filter):** Discards low-relevance results to prevent "hallucinations" based on noise.
  - **Any chunk? (If):** Checks if any valid data remains after filtering.
  - **Aggregate Matching Chunks:** Collects results for the current sub-query.
  - **Aggregate All Retrieval Results:** Consolidates results from all sub-queries into a single knowledge block for the final LLM.

#### 2.5 Answer Generation
**Overview:** Converts retrieved chunks into a structured, cited response.
- **Nodes Involved:** `Answer Generator AI Agent`, `Ollama Chat Model (Answer Generator — Qwen3:14b)`, `Postgres Chat Memory (RAG Answer)`, `Remove Think Tags (RAG Path)`, `Respond to Webhook`.
- **Node Details:**
  - **Answer Generator AI Agent:** Uses a strict system prompt requiring a 3-step format: Short Answer $\rightarrow$ Sources $\rightarrow$ Follow-up question.
  - **Ollama Chat Model:** Uses `qwen3:14b` with a low temperature (0.3) for precision.
  - **Postgres Chat Memory:** Maintains session context for the RAG path.
  - **Remove Think Tags (Code):** Strips reasoning blocks before returning the final string to the user.
  - **Respond to Webhook:** Sends the final cleaned output back to the requester.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | Webhook | Entry point | - | Understand Request | Overview |
| Understand Request | LLM Chain | Query Planner | Webhook | JSON Formatter | Classification |
| Ollama Chat Model (Classifier) | Ollama Model | LLM Engine (7B) | - | Understand Request | Classification |
| JSON Formatter | Code | Output Sanitizer | Understand Request | Switch | Classification |
| Switch | Switch | Path Router | JSON Formatter | Small Talk AI / Split Sub-Queries | Classification |
| Small Talk AI Agent | AI Agent | Conversationalist | Switch | Remove Think Tags (Small Talk) | Small Talk |
| Ollama Chat Model (Small Talk) | Ollama Model | LLM Engine (14B) | - | Small Talk AI Agent | Small Talk |
| Postgres Chat Memory (Small Talk)| Postgres Memory | Session Storage | - | Small Talk AI Agent | Small Talk |
| Remove Think Tags (Small Talk) | Code | Text Cleaning | Small Talk AI Agent | Respond to Webhook | Think Tags Warning |
| Split Sub-Queries | Split Out | Array to Items | Switch | Loop Over Sub-Queries | RAG Retrieval |
| Loop Over Sub-Queries | Split in Batches | Iteration | Split Sub-Queries | PGVector / Aggregate All | RAG Retrieval |
| PGVector Store | Vector Store | Vector Search | Loop Over Sub-Queries | Clean RAG output | RAG Retrieval |
| Ollama Embeddings (BGE-M3) | Embeddings | Vectorization | - | PGVector Store | RAG Retrieval |
| Clean RAG output | Set | Metadata Mapping | PGVector Store | Keep score over 0.4 | RAG Retrieval |
| Keep score over 0.4 | Filter | Quality Control | Clean RAG output | Any chunk? | RAG Retrieval |
| Any chunk? | If | Validation | Keep score over 0.4 | Aggregate Match / Say no match | RAG Retrieval |
| Say no chunk match | Set | Fallback Text | Any chunk? | Prepare loop output | RAG Retrieval |
| Aggregate Matching Chunks | Aggregate | Per-query Collection | Any chunk? | Prepare loop output | RAG Retrieval |
| Prepare loop output | Set | Loop State Manager | Say no match / Aggregate Match | Loop Over Sub-Queries | RAG Retrieval |
| Aggregate All Retrieval Results | Aggregate | Global Collection | Loop Over Sub-Queries | Answer Generator AI Agent | RAG Retrieval |
| Answer Generator AI Agent | AI Agent | Final Synthesizer | Aggregate All | Remove Think Tags (RAG) | Answer Generation |
| Ollama Chat Model (Answer Gen) | Ollama Model | LLM Engine (14B) | - | Answer Generator AI Agent | Answer Generation |
| Postgres Chat Memory (RAG) | Postgres Memory | Session Storage | - | Answer Generator AI Agent | Answer Generation |
| Remove Think Tags (RAG Path) | Code | Text Cleaning | Answer Generator AI Agent | Respond to Webhook | Think Tags Warning |
| Respond to Webhook | Respond to Webhook | Final Response | Remove Think Tags (Both) | - | Answer Generation |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Environment Setup
1. **Ollama:** Install Ollama and run:
   - `ollama pull qwen2.5:7b`
   - `ollama pull qwen3:14b`
   - `ollama pull bge-m3:latest`
2. **Database:** Install PostgreSQL and enable the `pgvector` extension. Create a `vector_store` table with columns for `content` and `metadata`.

#### Step 2: Input and Classification
1. Create a **Webhook** node (POST, path: `rag-chatbot`, Response Mode: `Response Node`).
2. Connect to an **LLM Chain** ("Understand Request").
   - Attach an **Ollama Chat Model** node (`qwen2.5:7b`).
   - Set the system prompt to classify input and generate a JSON array of `queries`.
3. Connect to a **Code** node ("JSON Formatter") to parse the LLM string into JSON using `JSON.parse()`.
4. Connect to a **Switch** node. 
   - Route 1: `is_question` is `false` $\rightarrow$ Path A.
   - Route 2: `is_question` is `true` $\rightarrow$ Path B.

#### Step 3: Path A - Small Talk
1. Create an **AI Agent** ("Small Talk AI Agent").
   - Attach an **Ollama Chat Model** (`qwen3:14b`).
   - Attach a **Postgres Chat Memory** node (Table: `chat_histories`, Session Key: `{{$json.body.session_id}}`).
2. Connect to a **Code** node to remove `<think>` tags using `.replace(/[\\s\\S]*?<\\/think>/gi, '')`.
3. Connect to **Respond to Webhook**.

#### Step 4: Path B - RAG Retrieval
1. Use a **Split Out** node to split the `queries` array into individual items.
2. Use a **Split in Batches** node ("Loop Over Sub-Queries").
3. Inside the loop, add a **PGVector Store** node (Mode: Load).
   - Attach **Ollama Embeddings** (`bge-m3:latest`).
   - Configure the table name `vector_store` and content column `content`.
4. Add a **Set** node ("Clean RAG output") to format the output as: `Chunk content`, `Resource File name`, and `Retrieval relevance score`.
5. Add a **Filter** node to keep only items where `Retrieval relevance score > 0.4`.
6. Use an **If** node to check if `Chunk content` exists.
   - **True:** Use an **Aggregate** node to collect results for the current query.
   - **False:** Use a **Set** node to write "No chunks reached the relevance threshold".
7. Use a **Set** node ("Prepare loop output") to save the query and its results, then loop back to the "Split in Batches" node.
8. After the loop finishes, use an **Aggregate** node to combine all retrieval results into one string.

#### Step 5: Final Answer Generation
1. Create an **AI Agent** ("Answer Generator AI Agent").
   - Attach **Ollama Chat Model** (`qwen3:14b`).
   - Attach **Postgres Chat Memory**.
   - **System Prompt:** Instruct the agent to provide a short answer, a list of sources, and a follow-up question.
2. Connect to a **Code** node to remove `<think>` tags.
3. Connect to **Respond to Webhook**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Qwen3 reasoning models output `<think>` blocks. If using non-reasoning models, the "Remove Think Tags" nodes can be deleted. | Model behavior specific to Qwen3 |
| Knowledge base must be populated via a separate ingestion workflow with metadata `title` and `file_path`. | Pre-requisite for RAG |
| Host URL for Ollama is typically `http://localhost:11434`. | Credential Configuration |