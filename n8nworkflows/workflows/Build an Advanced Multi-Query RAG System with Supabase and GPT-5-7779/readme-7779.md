Build an Advanced Multi-Query RAG System with Supabase and GPT-5

https://n8nworkflows.xyz/workflows/build-an-advanced-multi-query-rag-system-with-supabase-and-gpt-5-7779


# Build an Advanced Multi-Query RAG System with Supabase and GPT-5

### 1. Workflow Overview

This workflow implements an advanced Retrieval-Augmented Generation (RAG) system tailored for answering biology course-related questions through multi-query processing. It leverages a combination of an AI agent, a vector search engine (Supabase), and OpenAI's GPT-5 language model to produce high-quality, context-aware answers grounded strictly in a curated knowledge base.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Captures user chat input via a webhook and triggers the AI agent.
- **1.2 AI Agent Processing:** Uses a LangChain AI agent configured with a custom system prompt to:
  - Decompose complex queries into multiple sub-queries (up to 5).
  - Call a sub-workflow ("RAG sub-workflow") tool to retrieve relevant information chunks.
  - Reason critically over retrieved content using a "Think" tool to synthesize the final answer.
- **1.3 RAG Sub-Workflow (Query Handling and Filtering):** Processes the array of queries by:
  - Splitting them into individual queries.
  - Performing vector similarity search via Supabase for each query.
  - Filtering retrieved chunks by relevance score (>0.4).
  - Aggregating filtered chunks.
  - Returning the refined knowledge base content back to the AI agent.
- **1.4 Memory and Language Model Integration:** Incorporates short-term memory buffer and OpenAI’s GPT-5 Mini chat model to maintain conversational context and generate responses.
- **1.5 Output Preparation:** Aggregates results or handles no-match scenarios for final output delivery.

This modular design allows scalable, high-precision knowledge retrieval and answer generation, especially useful for complex educational or domain-specific AI assistants.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives user messages via a public webhook and triggers the AI agent.
- **Nodes Involved:**
  - `When chat message received`
- **Node Details:**
  - **When chat message received**
    - *Type:* LangChain Chat Trigger Node
    - *Role:* Entry point for user chat messages.
    - *Configuration:*
      - Public webhook enabled for external access.
      - Response mode set to return output from the last executed node.
    - *Inputs:* External webhook call with user query.
    - *Outputs:* Triggers the `AI Agent` node.
    - *Failure Modes:* Webhook connectivity issues, malformed input.
    - *Version:* 1.3

#### 2.2 AI Agent Processing

- **Overview:** Decomposes user queries, orchestrates sub-query retrieval, and synthesizes answers based on retrieved knowledge.
- **Nodes Involved:**
  - `AI Agent`
  - `Query knowledge base`
  - `Think`
  - `Simple Memory`
  - `OpenAI Chat Model`
- **Node Details:**

  - **AI Agent**
    - *Type:* LangChain AI Agent Node
    - *Role:* Main orchestrator that interprets user queries and manages sub-queries.
    - *Configuration:*
      - Custom system message specifying the assistant is biology-course-specific.
      - Instructions to decompose complex queries into 1–5 sub-queries.
      - Mandates the use of the "Query knowledge base" tool to fetch relevant chunks.
      - Use of the "Think" tool to reason critically before answering.
      - Transparent fallback if information is out of scope.
      - Streaming disabled.
    - *Inputs:* Trigger from `When chat message received`.
    - *Outputs:* Calls `Query knowledge base` tool workflow.
    - *Failure Modes:* Expression errors, logic errors in query decomposition, tool invocation failures.
    - *Version:* 2.2

  - **Query knowledge base**
    - *Type:* LangChain Tool Workflow Node
    - *Role:* Invokes the RAG sub-workflow to fetch knowledge base content using multiple queries.
    - *Configuration:*
      - Calls sub-workflow "TEMPLATE RAG with Supabase and GPT5" (workflow ID: `c9FlK6mLuWAwqLsP`).
      - Passes an array of queries formatted as JSON objects.
      - Explicit instructions to always pass queries as arrays (even if single).
    - *Inputs:* Receives queries array from `AI Agent`.
    - *Outputs:* Returns retrieved knowledge chunks.
    - *Failure Modes:* Sub-workflow invocation failure, incorrect query formatting.
    - *Version:* 2.2

  - **Think**
    - *Type:* LangChain Tool Think Node
    - *Role:* Performs concise (max 50 words) critical analysis on retrieved content vis-à-vis the original query.
    - *Configuration:* Token-efficient reasoning to verify content accuracy before final response.
    - *Inputs:* Output from `Query knowledge base`.
    - *Outputs:* Feeds reasoning back to `AI Agent`.
    - *Failure Modes:* Token limit exceeded, reasoning errors.
    - *Version:* 1.1

  - **Simple Memory**
    - *Type:* LangChain Memory Buffer Window Node
    - *Role:* Maintains conversational context by storing the last 8 interactions.
    - *Inputs:* Connected to and from `AI Agent`.
    - *Outputs:* Provides context to `AI Agent` for coherent conversation.
    - *Failure Modes:* Memory overflow, context mismatch.
    - *Version:* 1.3

  - **OpenAI Chat Model**
    - *Type:* LangChain OpenAI Chat Model Node
    - *Role:* Language model backend for AI Agent.
    - *Configuration:* Uses GPT-5 Mini model.
    - *Credentials:* OpenAI API credential ("Duv's OpenAI").
    - *Inputs:* Receives prompts from `AI Agent`.
    - *Outputs:* Generates text responses.
    - *Failure Modes:* API authentication errors, rate limits, model unavailability.
    - *Version:* 1.2

#### 2.3 RAG Sub-Workflow: Query Handling and Filtering

- **Overview:** Processes each sub-query individually to retrieve relevant content chunks from Supabase vector store, filters by relevance score, aggregates results, and prepares output.
- **Nodes Involved:**
  - `RAG sub-workflow` (external workflow invoked by `Query knowledge base`)
  - `Split Out`
  - `Loop Over Items1`
  - `Supabase Vector Store1`
  - `Embeddings OpenAI1`
  - `Clean RAG output`
  - `Keep score over 0.4`
  - `Any chunk?`
  - `Aggregate chunks`
  - `Say no chunk match`
  - `Aggregate items`
  - `Prepare loop output`
- **Node Details:**

  - **RAG sub-workflow**
    - *Type:* Execute Workflow Trigger Node
    - *Role:* Receives queries array input and starts processing.
    - *Inputs:* Array of queries.
    - *Outputs:* Passes queries to `Split Out`.
    - *Failure Modes:* Sub-workflow execution failure.
    - *Version:* 1.1

  - **Split Out**
    - *Type:* Split Out Node
    - *Role:* Splits the `queries` array into individual query items.
    - *Inputs:* From `RAG sub-workflow`.
    - *Outputs:* Single query to `Loop Over Items1`.
    - *Version:* 1

  - **Loop Over Items1**
    - *Type:* Split In Batches Node
    - *Role:* Iterates over individual queries one by one.
    - *Inputs:* From `Split Out`.
    - *Outputs:* Parallel outputs:
      - To `Aggregate items` (collects queries for later aggregation).
      - To `Supabase Vector Store1` (performs vector search).
    - *Failure Modes:* Batch processing errors.
    - *Version:* 3

  - **Supabase Vector Store1**
    - *Type:* LangChain Vector Store Supabase Node
    - *Role:* Executes vector similarity search for each query.
    - *Configuration:*
      - Mode: Load (query mode).
      - Query prompt: bound dynamically to current query text.
      - Uses Supabase table "documents".
      - Uses Supabase API credential ("Supabase account 2").
    - *Inputs:* Single query from `Loop Over Items1`.
    - *Outputs:* Retrieves documents with metadata and scores.
    - *Failure Modes:* API errors, credential issues, empty results.
    - *Version:* 1.3

  - **Embeddings OpenAI1**
    - *Type:* LangChain OpenAI Embeddings Node
    - *Role:* Generates embeddings for queries if needed (used internally by Supabase node).
    - *Credentials:* OpenAI API ("1 - Plan A's OpenAI").
    - *Inputs:* Query text.
    - *Outputs:* Embeddings vector.
    - *Failure Modes:* API errors, rate limits.
    - *Version:* 1.2

  - **Clean RAG output**
    - *Type:* Set Node
    - *Role:* Extracts and reformats relevant fields from each retrieved document chunk.
    - *Configuration:*
      - Maps `Chunk content` to document page content.
      - Creates `Chunk metadata` object with:
        - Resource chapter name (from document metadata).
        - Retrieval relevance score rounded to two decimals.
    - *Inputs:* Output from `Supabase Vector Store1`.
    - *Outputs:* Cleaned chunk data to `Keep score over 0.4`.
    - *Failure Modes:* Missing fields, data format issues.
    - *Version:* 3.4

  - **Keep score over 0.4**
    - *Type:* Filter Node
    - *Role:* Filters chunks to keep only those with relevance score > 0.4.
    - *Inputs:* From `Clean RAG output`.
    - *Outputs:* Passes filtered chunks to `Any chunk?`.
    - *Failure Modes:* Filtering logic errors.
    - *Version:* 2.2
    - *Note:* Always outputs data even if empty.

  - **Any chunk?**
    - *Type:* If Node
    - *Role:* Checks if any chunk content exists after filtering.
    - *Condition:* Existence of non-empty "Chunk content".
    - *Outputs:*
      - True: To `Aggregate chunks`.
      - False: To `Say no chunk match`.
    - *Failure Modes:* Expression evaluation errors.
    - *Version:* 2.2

  - **Aggregate chunks**
    - *Type:* Aggregate Node
    - *Role:* Combines all filtered chunks for the current query into one array under "All chunks for this question".
    - *Inputs:* From `Any chunk?` (true branch).
    - *Outputs:* To `Prepare loop output`.
    - *Failure Modes:* Data aggregation errors.
    - *Version:* 1

  - **Say no chunk match**
    - *Type:* Set Node
    - *Role:* Sets a fallback message when no chunks exceed the relevance threshold.
    - *Output Field:* `Retrieval output` with explanatory string.
    - *Inputs:* From `Any chunk?` (false branch).
    - *Outputs:* To `Prepare loop output`.
    - *Version:* 3.4

  - **Aggregate items**
    - *Type:* Aggregate Node
    - *Role:* Aggregates all original queries again for final output preparation.
    - *Inputs:* From `Loop Over Items1`.
    - *Outputs:* Not directly connected, used internally.
    - *Version:* 1

  - **Prepare loop output**
    - *Type:* Set Node
    - *Role:* Prepares the output object for each query iteration, including:
      - Original query string.
      - JSON stringified chunks returned or fallback message.
    - *Inputs:* From `Aggregate chunks` or `Say no chunk match`.
    - *Outputs:* Loops back to `Loop Over Items1` for processing next query or returning aggregated results.
    - *Version:* 3.4

#### 2.4 Memory and Language Model Integration

- **Overview:** Maintains conversational history and uses a GPT-5 Mini model for natural language generation within the AI Agent.
- **Nodes Involved:**
  - `Simple Memory`
  - `OpenAI Chat Model`
- **Node Details:** Covered in section 2.2.

#### 2.5 Output Preparation

- **Overview:** Aggregates final results from all sub-queries into a coherent response, or returns a no-match message.
- **Nodes Involved:**
  - `Aggregate chunks`
  - `Prepare loop output`
  - `Say no chunk match`
- **Node Details:** Covered in section 2.3.

---

### 3. Summary Table

| Node Name                | Node Type                                  | Functional Role                         | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                             |
|--------------------------|--------------------------------------------|---------------------------------------|------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger      | Entry point webhook for user queries  | External webhook       | AI Agent                  |                                                                                                        |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent             | Orchestrates query decomposition and response synthesis | When chat message received, OpenAI Chat Model, Simple Memory, Think, Query knowledge base | Query knowledge base, Think, Simple Memory, OpenAI Chat Model | # AI agent                                                                                             |
| Query knowledge base      | @n8n/n8n-nodes-langchain.toolWorkflow      | Calls RAG sub-workflow with queries   | AI Agent               | AI Agent                  | # Sub-workflow, tool for agent                                                                          |
| Think                    | @n8n/n8n-nodes-langchain.toolThink         | Critical reasoning on retrieved content | Query knowledge base   | AI Agent                  | # AI agent                                                                                             |
| Simple Memory            | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores conversation context           | AI Agent               | AI Agent                  | # AI agent                                                                                             |
| OpenAI Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Language model backend for AI Agent   | AI Agent               | AI Agent                  | # AI agent                                                                                             |
| RAG sub-workflow          | n8n-nodes-base.executeWorkflowTrigger      | Splits queries and handles retrieval  | Query knowledge base    | Split Out                 | # Sub-workflow, tool for agent                                                                          |
| Split Out                | n8n-nodes-base.splitOut                     | Splits queries array into single items | RAG sub-workflow       | Loop Over Items1          | # Sub-workflow, tool for agent                                                                          |
| Loop Over Items1          | n8n-nodes-base.splitInBatches               | Iterates over queries                  | Split Out              | Aggregate items, Supabase Vector Store1 | # Sub-workflow, tool for agent                                                                          |
| Supabase Vector Store1    | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Retrieves relevant chunks via vector search | Loop Over Items1       | Clean RAG output          | # Sub-workflow, tool for agent                                                                          |
| Embeddings OpenAI1        | @n8n/n8n-nodes-langchain.embeddingsOpenAi  | Provides embeddings for vector search | Supabase Vector Store1  | Supabase Vector Store1    | # Sub-workflow, tool for agent                                                                          |
| Clean RAG output          | n8n-nodes-base.set                          | Extracts and formats chunk content    | Supabase Vector Store1  | Keep score over 0.4       | ## Filtering system Only keeping chunks that have a score >0.4                                         |
| Keep score over 0.4       | n8n-nodes-base.filter                       | Filters chunks by relevance score      | Clean RAG output       | Any chunk?                | ## Filtering system Only keeping chunks that have a score >0.4                                         |
| Any chunk?                | n8n-nodes-base.if                           | Checks if any relevant chunk exists    | Keep score over 0.4    | Aggregate chunks, Say no chunk match | ## Filtering system Only keeping chunks that have a score >0.4                                         |
| Aggregate chunks          | n8n-nodes-base.aggregate                     | Aggregates filtered chunks             | Any chunk? (true)      | Prepare loop output       | ## Filtering system Only keeping chunks that have a score >0.4                                         |
| Say no chunk match        | n8n-nodes-base.set                          | Sets fallback message if no chunk matches | Any chunk? (false)     | Prepare loop output       |                                                                                                        |
| Aggregate items           | n8n-nodes-base.aggregate                     | Aggregates original queries            | Loop Over Items1       |                           |                                                                                                        |
| Prepare loop output       | n8n-nodes-base.set                          | Prepares final output for each query   | Aggregate chunks, Say no chunk match | Loop Over Items1          |                                                                                                        |
| Sticky Note               | n8n-nodes-base.stickyNote                    | Notes and documentation                | -                      | -                         | # Advanced Multi-Query RAG Agent with usage instructions and architecture overview                      |
| Sticky Note1              | n8n-nodes-base.stickyNote                    | Notes the AI Agent section             | -                      | -                         | # AI agent                                                                                             |
| Sticky Note2              | n8n-nodes-base.stickyNote                    | Notes the sub-workflow/tool for agent  | -                      | -                         | # Sub-workflow, tool for agent                                                                          |
| Sticky Note3              | n8n-nodes-base.stickyNote                    | Filtering system explanation           | -                      | -                         | ## Filtering system Only keeping chunks that have a score >0.4                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `When chat message received` Node**
   - Type: LangChain Chat Trigger
   - Set Public webhook enabled
   - Response mode: "lastNode"
   - Save webhook URL for external calls

2. **Create `AI Agent` Node**
   - Type: LangChain AI Agent
   - System message:
     ```
     You are a helpful assistant that answers based on a biology course.

     For that, you always start by calling the tool "Query knowledge base" to send an array of 1 to 5 questions that are relevant to ask to the RAG knowledge base that contains all the content of the course and get as an output all chunks that seem to help to craft the final answer. The more the user query is complex, the more you will break it down into sub-queries (up to 5).

     From there, use the Think tool to critically analyse the initial user query and the content you've retrieved from the knowledge retrieval tool and reason to prepare the best answer possible, challenge the content to be sure that you actually have the right information to be able to respond.

     Only answer based on the course content that you get from using the tool, if you receive any question outside that scope, redirect the conversation, if you don't have the right information to answer, be transparent and say so - don't try to reply anyway with general knowledge.
     ```
   - Disable streaming
   - Connect `When chat message received` → `AI Agent`

3. **Create `Query knowledge base` Node**
   - Type: LangChain Tool Workflow
   - Link to sub-workflow: "TEMPLATE RAG with Supabase and GPT5" (or create as per section 4.4)
   - Configure to accept `queries` as array input in JSON format
   - Set mapping to pass the `queries` array from the AI Agent
   - Connect `AI Agent` → `Query knowledge base`

4. **Create `Think` Node**
   - Type: LangChain Tool Think
   - Description:
     ```
     Use this tool after you got the output of the knowledge retrieval tool to critically analyse the initial user query and the content you've retrieved from the knowledge retrieval tool and reason to prepare the best answer possible, challenge the content to be sure that you actually have the right information to be able to respond.

     Be very token efficient when using this tool, write 50 words max which is enough to reason.
     ```
   - Connect `Query knowledge base` → `Think`
   - Connect `Think` → `AI Agent` (ai_tool input)

5. **Create `Simple Memory` Node**
   - Type: LangChain Memory Buffer Window
   - Context window length: 8
   - Connect `AI Agent` → `Simple Memory` (ai_memory input)
   - Connect `Simple Memory` → `AI Agent` (ai_memory output)

6. **Create `OpenAI Chat Model` Node**
   - Type: LangChain OpenAI Chat Model
   - Model: "gpt-5-mini"
   - Credentials: Configure OpenAI API credentials (e.g., "Duv's OpenAI")
   - Connect `AI Agent` → `OpenAI Chat Model` (ai_languageModel input)
   - Connect `OpenAI Chat Model` → `AI Agent` (ai_languageModel output)

7. **Create RAG Sub-Workflow**

   - **Create `RAG sub-workflow` with input parameter:**
     - `queries` (array)

   - Inside RAG sub-workflow:

     1. **Add `Split Out` Node**
        - Type: Split Out
        - Field to split: `queries`
        - Connect input to trigger node input

     2. **Add `Loop Over Items1` Node**
        - Type: Split In Batches
        - Connect `Split Out` → `Loop Over Items1`

     3. **Add `Aggregate items` Node**
        - Type: Aggregate
        - Aggregate all item data
        - Connect `Loop Over Items1` (main output 1) → `Aggregate items`

     4. **Add `Supabase Vector Store1` Node**
        - Type: LangChain Vector Store Supabase
        - Mode: Load (query mode)
        - Query prompt: `={{ $json.query }}`
        - Table name: "documents"
        - Credentials: Configure Supabase API (e.g., "Supabase account 2")
        - Connect `Loop Over Items1` (main output 2) → `Supabase Vector Store1`

     5. **Add `Embeddings OpenAI1` Node**
        - Type: LangChain Embeddings OpenAI
        - Credentials: OpenAI API (e.g., "1 - Plan A's OpenAI")
        - Connect `Embeddings OpenAI1` → `Supabase Vector Store1` (ai_embedding input)

     6. **Add `Clean RAG output` Node**
        - Type: Set Node
        - Assignments:
          - `Chunk content` = `={{ $json.document.pageContent }}`
          - `Chunk metadata` = JSON object with:
            - `Resource chapter name` = `{{ $json.document.metadata['Chapter name'] }}`
            - `Retrieval relevance score` = rounded `{{ $json.score.round(2) }}`
        - Connect `Supabase Vector Store1` → `Clean RAG output`

     7. **Add `Keep score over 0.4` Node**
        - Type: Filter Node
        - Condition: Keep only if `Chunk metadata.Retrieval relevance score` > 0.4
        - Always output data (even if empty)
        - Connect `Clean RAG output` → `Keep score over 0.4`

     8. **Add `Any chunk?` Node**
        - Type: If Node
        - Condition: Check if `Chunk content` exists and is non-empty
        - True branch → `Aggregate chunks`
        - False branch → `Say no chunk match`
        - Connect `Keep score over 0.4` → `Any chunk?`

     9. **Add `Aggregate chunks` Node**
        - Type: Aggregate Node
        - Aggregate all item data into `All chunks for this question`
        - Connect `Any chunk?` (true) → `Aggregate chunks`

     10. **Add `Say no chunk match` Node**
         - Type: Set Node
         - Set field `Retrieval output` with message: "No chunks reached the relevance threshold, the knowledge base was unable to provide information that would be helpful to answer this question."
         - Connect `Any chunk?` (false) → `Say no chunk match`

     11. **Add `Prepare loop output` Node**
         - Type: Set Node
         - Assign:
           - `Query to the knowledge base` = first query string from `Loop Over Items1`
           - `Chunks returned` = JSON stringified results from either `Aggregate chunks` or `Say no chunk match`
         - Connect `Aggregate chunks` → `Prepare loop output`
         - Connect `Say no chunk match` → `Prepare loop output`
         - Connect `Prepare loop output` → `Loop Over Items1` (to continue loop or finish)

8. **Connect `Query knowledge base` Node to the RAG Sub-Workflow**
   - Ensure it passes the `queries` array input into the sub-workflow.

9. **Test the entire workflow with sample queries**
   - Example query: "Explain the process of photosynthesis."
   - Verify the multi-query decomposition, chunk retrieval, filtering, reasoning, and final answer.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow implements an advanced multi-query RAG agent designed to handle complex questions by decomposing them into sub-queries, retrieving relevant information chunks from Supabase vector store, filtering low-relevance chunks, and reasoning critically before answering. It is optimized for domain-specific knowledge bases, here exemplified by a biology course.            | Sticky Note on main workflow ("Advanced Multi-Query RAG Agent")                                       |
| To customize for other domains, edit the AI Agent's system prompt to reflect the target knowledge base and ensure that the Supabase dataset contains relevant documents.                                                                                                                                                                                                    | Sticky Note instructions and AI Agent system prompt                                                  |
| The default relevance score filter (>0.4) can be adjusted in the sub-workflow filter node to tune the quality vs. quantity of retrieved chunks.                                                                                                                                                                                                                                | Sticky Note3 near filtering nodes                                                                     |
| Required Credentials: Supabase API keys with access to the vector store table "documents"; OpenAI API keys for embeddings generation and GPT-5 Mini chat completions.                                                                                                                                                                                                        | Credential configurations in nodes                                                                    |
| For further information on RAG architectures and LangChain integrations, consult the official n8n and LangChain documentation.                                                                                                                                                                                                                                              | n8n docs: https://docs.n8n.io/ , LangChain docs: https://python.langchain.com/en/latest/               |

---

This detailed analysis and reconstruction guide provides a clear understanding of the workflow’s structure and logic, enabling advanced users and AI agents to reproduce, customize, or extend this advanced multi-query RAG system efficiently.