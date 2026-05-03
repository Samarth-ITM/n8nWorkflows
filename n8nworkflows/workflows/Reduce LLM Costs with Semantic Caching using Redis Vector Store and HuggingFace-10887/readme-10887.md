Reduce LLM Costs with Semantic Caching using Redis Vector Store and HuggingFace

https://n8nworkflows.xyz/workflows/reduce-llm-costs-with-semantic-caching-using-redis-vector-store-and-huggingface-10887


# Reduce LLM Costs with Semantic Caching using Redis Vector Store and HuggingFace

### 1. Workflow Overview

This workflow implements a **semantic caching system** for reducing Large Language Model (LLM) operational costs and improving response times by caching semantically similar queries and their responses. It is designed for chat applications where user queries are processed, checked against a Redis vector store cache for similarity, and either served instantly from cache or forwarded to an LLM for fresh generation.

The workflow is logically grouped into these blocks:

- **1.1 Input Reception:** Receiving chat messages via a webhook trigger.
- **1.2 Semantic Similarity Search:** Using HuggingFace embeddings and Redis vector store to find similar cached queries.
- **1.3 Cache Analysis and Decision:** Analyzing similarity scores to determine cache hits or misses.
- **1.4 Cache Hit Response:** Returning cached responses immediately for similar queries.
- **1.5 Cache Miss Processing:** Invoking an LLM agent (GPT-4.1-mini) to generate a new response.
- **1.6 Cache Storage:** Embedding and storing new query-response pairs back into Redis cache.
- **1.7 Chat Memory Management:** Maintaining conversational context using Redis chat memory.
- **1.8 Response Delivery:** Sending the final response back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming chat messages from users to initiate the workflow.
- **Nodes Involved:** 
  - `When chat message received`
- **Node Details:**
  - Type: `chatTrigger` (Langchain)
  - Role: Entry point webhook node that triggers workflow on incoming chat message.
  - Configuration: Uses webhook with response mode set to `responseNodes` for asynchronous response handling.
  - Connections: Outputs to `Embeddings HuggingFace Inference` and `Check for similar prompts`.
  - Failure Cases: Webhook timeout, malformed input, or connectivity issues.
  - Notes: This node requires webhook setup and exposure for chat interface integration.

#### 2.2 Semantic Similarity Search

- **Overview:** Converts incoming query text to vector embeddings and queries Redis vector store to find semantically similar cached entries.
- **Nodes Involved:** 
  - `Embeddings HuggingFace Inference`
  - `Check for similar prompts`
- **Node Details:**

  - **Embeddings HuggingFace Inference**
    - Type: `embeddingsHuggingFaceInference` (Langchain)
    - Role: Transforms chat input text into vector embeddings using HuggingFace models.
    - Configuration: Default options; uses HuggingFace inference API.
    - Input: Text from `When chat message received`
    - Output: Embeddings for Redis vector search.
    - Failure Cases: API rate limits, invalid credentials, or service downtime.

  - **Check for similar prompts**
    - Type: `vectorStoreRedis` (Langchain)
    - Role: Performs vector similarity search in Redis index `chat_cache`.
    - Configuration: Mode set to `load`, prompt set dynamically from incoming chat input.
    - Input: Embeddings from `Embeddings HuggingFace Inference`
    - Output: List of similar documents with similarity scores.
    - Failure Cases: Redis connection failures, missing index, query errors.

#### 2.3 Cache Analysis and Decision

- **Overview:** Filters Redis search results by similarity threshold and decides if the query is a cache hit or miss.
- **Nodes Involved:** 
  - `Analyze results from store`
  - `Is this a cache hit?`
- **Node Details:**

  - **Analyze results from store**
    - Type: `code` (JavaScript)
    - Role: Filters documents with similarity scores below threshold (default 0.3), selects best match.
    - Configuration: Custom JS code with adjustable `distanceThreshold`.
    - Input: All documents from `Check for similar prompts`
    - Output: Single best matching document or empty if no matches.
    - Failure Cases: Code errors, empty input, unexpected data shapes.

  - **Is this a cache hit?**
    - Type: `if`
    - Role: Checks if a document exists in filtered results to branch cache hit vs miss.
    - Configuration: Condition tests existence of `document` field.
    - Input: Output from `Analyze results from store`
    - Outputs:
      - True branch: Cache hit → `Respond to Chat (from semantic cache)`
      - False branch: Cache miss → `LLM Agent`
    - Failure Cases: Expression evaluation errors.

#### 2.4 Cache Hit Response

- **Overview:** Sends cached response back to the user immediately, avoiding LLM call.
- **Nodes Involved:** 
  - `Respond to Chat (from semantic cache)`
- **Node Details:**
  - Type: `chat`
  - Role: Sends response stored in cache (`document.metadata.reply`) back to the user.
  - Configuration: Message set dynamically from cached metadata.
  - Input: Cache hit from `Is this a cache hit?`
  - Output: Ends the workflow response chain here.
  - Failure Cases: Missing or malformed cached data.

#### 2.5 Cache Miss Processing

- **Overview:** Invokes LLM agent to generate a new answer for queries not found in cache.
- **Nodes Involved:** 
  - `LLM Agent`
  - `OpenAI Chat Model`
  - `Redis Chat Memory`
- **Node Details:**

  - **LLM Agent**
    - Type: `agent` (Langchain)
    - Role: Processes user input with conversation memory to generate a new response.
    - Configuration:
      - Text input from `When chat message received`
      - Max 10 iterations for agent reasoning
      - System message defines assistant behavior
    - Inputs:
      - `OpenAI Chat Model` for LLM backend
      - `Redis Chat Memory` for conversation context
    - Outputs: Response text and metadata to cache storage.
    - Failure Cases: LLM API errors, memory retrieval failures.

  - **OpenAI Chat Model**
    - Type: `lmChatOpenAi`
    - Role: Provides GPT-4.1-mini LLM capabilities.
    - Configuration: Model set to `gpt-4.1-mini`.
    - Inputs: From `LLM Agent`
    - Failure Cases: API authentication issues, rate limits.

  - **Redis Chat Memory**
    - Type: `memoryRedisChat`
    - Role: Stores and fetches conversation history in Redis for context.
    - Inputs: Used by `LLM Agent`
    - Failure Cases: Redis connectivity.

#### 2.6 Cache Storage

- **Overview:** Embeds the new response, attaches metadata, and inserts the query-response pair into Redis cache for future reuse.
- **Nodes Involved:** 
  - `Recursive Character Text Splitter`
  - `Add response as metadata`
  - `Embeddings HuggingFace Inference1`
  - `Store entry in cache`
- **Node Details:**

  - **Recursive Character Text Splitter**
    - Type: `textSplitterRecursiveCharacterTextSplitter`
    - Role: Splits text into manageable chunks for embedding.
    - Input: Response from `LLM Agent`
    - Output: Split text for metadata addition.
    - Failure Cases: Malformed text input.

  - **Add response as metadata**
    - Type: `documentDefaultDataLoader`
    - Role: Attaches the LLM-generated reply as metadata field `reply` to the document.
    - Input: Output from text splitter and original query.
    - Output: Document ready for vector storage.
    - Failure Cases: Data mapping errors.

  - **Embeddings HuggingFace Inference1**
    - Type: `embeddingsHuggingFaceInference`
    - Role: Converts the document text with metadata into vector embeddings.
    - Input: Document with metadata.
    - Output: Embeddings for Redis insertion.
    - Failure Cases: API errors or rate limits.

  - **Store entry in cache**
    - Type: `vectorStoreRedis`
    - Role: Inserts or overwrites the new query-response vector document into Redis index `chat_cache`.
    - Configuration: Mode set to `insert`, overwriting allowed.
    - Input: Embeddings from previous node.
    - Output: Connected to `Respond to Chat (from LLM)`
    - Failure Cases: Redis write errors, index missing.

#### 2.7 Response Delivery

- **Overview:** Sends the freshly generated LLM response back to the user.
- **Nodes Involved:** 
  - `Respond to Chat (from LLM)`
- **Node Details:**
  - Type: `chat`
  - Role: Sends the `metadata.reply` message generated by the LLM to the chat interface.
  - Input: From `Store entry in cache`
  - Failure Cases: Message formatting or delivery errors.

#### 2.8 Sticky Notes

- **Overview:** Provide documentation, tuning tips, and best practices embedded in the workflow.
- **Nodes Involved:** 
  - `Sticky Note`
  - `Sticky Note2`
  - `Sticky Note3`
- **Content Highlights:**
  - Workflow purpose and architecture overview.
  - Cache tuning instructions, especially on `distanceThreshold` in similarity analysis.
  - Cost benefits of using embedding models and notes on performance.

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                              | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                      |
|--------------------------------|--------------------------------|----------------------------------------------|----------------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When chat message received      | chatTrigger                    | Entry webhook for incoming chat messages    | -                                | Embeddings HuggingFace Inference, Check for similar prompts | Try it out! This workflow implements an intelligent semantic caching system that reduces LLM costs and improves response times by caching similar queries. See detailed overview in the main sticky note. |
| Embeddings HuggingFace Inference| embeddingsHuggingFaceInference | Converts chat input text to embeddings       | When chat message received       | Check for similar prompts             |                                                                                                                 |
| Check for similar prompts       | vectorStoreRedis               | Searches Redis vector store for similar queries | When chat message received, Embeddings HuggingFace Inference | Analyze results from store              |                                                                                                                 |
| Analyze results from store      | code                          | Filters and selects best similarity match    | Check for similar prompts         | Is this a cache hit?                   | Tuning the Cache: Adjust `distanceThreshold` to control cache sensitivity. Lower threshold = stricter match.   |
| Is this a cache hit?            | if                            | Decides cache hit or miss based on similarity | Analyze results from store        | Respond to Chat (from semantic cache), LLM Agent |                                                                                                                 |
| Respond to Chat (from semantic cache) | chat                     | Returns cached response quickly                | Is this a cache hit? (true branch) | -                                    |                                                                                                                 |
| LLM Agent                      | agent                         | Generates new response using GPT-4.1-mini    | Is this a cache hit? (false branch), OpenAI Chat Model, Redis Chat Memory | Store entry in cache                  | Embedding model note: Using your own embedding model can reduce costs and improve performance.                  |
| OpenAI Chat Model              | lmChatOpenAi                  | Provides GPT-4.1-mini LLM backend             | LLM Agent                       | LLM Agent                            |                                                                                                                 |
| Redis Chat Memory              | memoryRedisChat               | Maintains conversation memory across chats    | LLM Agent                       | LLM Agent                            |                                                                                                                 |
| Store entry in cache           | vectorStoreRedis              | Inserts new query-response pair into Redis    | LLM Agent, Add response as metadata | Respond to Chat (from LLM)           |                                                                                                                 |
| Respond to Chat (from LLM)     | chat                         | Sends LLM-generated response to user          | Store entry in cache             | -                                    |                                                                                                                 |
| Recursive Character Text Splitter | textSplitterRecursiveCharacterTextSplitter | Splits LLM response text for embedding        | LLM Agent                      | Add response as metadata              |                                                                                                                 |
| Add response as metadata       | documentDefaultDataLoader     | Attaches reply metadata to document            | Recursive Character Text Splitter, When chat message received | Store entry in cache                   |                                                                                                                 |
| Embeddings HuggingFace Inference1 | embeddingsHuggingFaceInference | Creates embeddings for new cache entry         | Add response as metadata         | Store entry in cache                   |                                                                                                                 |
| Sticky Note                   | stickyNote                   | Overview and workflow purpose                  | -                              | -                                    | See section 5 for full note content                                                                             |
| Sticky Note2                  | stickyNote                   | Cache tuning guidance                           | -                              | -                                    | Tuning the Cache: Adjust `distanceThreshold` in Analyze results from store node                                 |
| Sticky Note3                  | stickyNote                   | Embedding model cost/performance note          | -                              | -                                    | Embedding model note: Using your own embedding model reduces costs and increases performance                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**
   - Type: `Langchain Chat Trigger`
   - Name: `When chat message received`
   - Configure webhook with response mode `responseNodes`.
   - This node receives user chat messages.

2. **Create HuggingFace Embeddings Node:**
   - Type: `Langchain Embeddings HuggingFace Inference`
   - Name: `Embeddings HuggingFace Inference`
   - Connect input from `When chat message received`.
   - Default options; no special configuration needed.
   - Requires HuggingFace API credentials configured in n8n.

3. **Create Redis Vector Store Search Node:**
   - Type: `Langchain Vector Store Redis`
   - Name: `Check for similar prompts`
   - Mode: `load`
   - Redis index: `chat_cache`
   - Set prompt input expression as `{{$json.chatInput}}`.
   - Connect AI embedding input from `Embeddings HuggingFace Inference`.
   - Connect main input from `When chat message received`.
   - Requires Redis credentials and Redis server running version 8.x+.

4. **Create JavaScript Code Node to Analyze Similarity:**
   - Type: `Code`
   - Name: `Analyze results from store`
   - Language: JavaScript
   - Code:
     ```js
     const distanceThreshold = 0.3;
     const allMatches = $input.all().filter(item => item.json.score < distanceThreshold);
     return allMatches.length > 1 ? allMatches.reduce((min, item) => item.json.score < min.json.score ? item : min) : allMatches;
     ```
   - Connect main input from `Check for similar prompts`.

5. **Create If Node to Check Cache Hit:**
   - Type: `If`
   - Name: `Is this a cache hit?`
   - Condition: Check if `document` field exists in JSON.
   - Connect main input from `Analyze results from store`.
   - True output branch: cache hit path.
   - False output branch: cache miss path.

6. **Create Chat Node to Respond from Cache:**
   - Type: `Langchain Chat`
   - Name: `Respond to Chat (from semantic cache)`
   - Message: Expression `{{$json.document.metadata.reply}}`
   - Connect main input from `Is this a cache hit?` true branch.

7. **Create OpenAI Chat Model Node:**
   - Type: `Langchain LmChatOpenAi`
   - Name: `OpenAI Chat Model`
   - Model: `gpt-4.1-mini`
   - Connect as AI language model input for agent node.
   - Configure OpenAI credentials with valid API key.

8. **Create Redis Chat Memory Node:**
   - Type: `Langchain Memory Redis Chat`
   - Name: `Redis Chat Memory`
   - Connect as AI memory input for agent node.
   - Requires Redis credentials.

9. **Create LLM Agent Node:**
   - Type: `Langchain Agent`
   - Name: `LLM Agent`
   - Text: Expression `{{$('When chat message received').item.json.chatInput}}`
   - Max iterations: 10
   - System message: "You are a helpful assistant helping out with generic questions. Reply to the user with your best answer and don't invent much."
   - Connect AI language model input from `OpenAI Chat Model`.
   - Connect AI memory input from `Redis Chat Memory`.
   - Connect main input from `Is this a cache hit?` false branch.

10. **Create Recursive Text Splitter Node:**
    - Type: `Langchain TextSplitter RecursiveCharacterTextSplitter`
    - Name: `Recursive Character Text Splitter`
    - Connect AI text splitter input from `LLM Agent`.

11. **Create Document Data Loader Node:**
    - Type: `Langchain Document Default Data Loader`
    - Name: `Add response as metadata`
    - Metadata: Add field `reply` with value `{{$json.output}}` from LLM Agent.
    - JSON Data: Expression `{{$('When chat message received').item.json.chatInput}}`
    - Connect AI document input from `Recursive Character Text Splitter`.

12. **Create Embeddings Node for Cache Storage:**
    - Type: `Langchain Embeddings HuggingFace Inference`
    - Name: `Embeddings HuggingFace Inference1`
    - Connect AI embedding input from `Add response as metadata`.

13. **Create Redis Vector Store Insert Node:**
    - Type: `Langchain Vector Store Redis`
    - Name: `Store entry in cache`
    - Mode: `insert`
    - Options: Enable `overwriteDocuments`
    - Redis index: `chat_cache`
    - Connect main input from `LLM Agent`.
    - Connect AI document input from `Add response as metadata`.
    - Connect AI embedding input from `Embeddings HuggingFace Inference1`.

14. **Create Chat Node to Respond from LLM:**
    - Type: `Langchain Chat`
    - Name: `Respond to Chat (from LLM)`
    - Message: Expression `{{$json.metadata.reply}}`
    - Connect main input from `Store entry in cache`.

15. **Connect Workflow Flow:**
    - `When chat message received` → `Embeddings HuggingFace Inference`
    - `When chat message received` → `Check for similar prompts`
    - `Embeddings HuggingFace Inference` → `Check for similar prompts` (AI embedding)
    - `Check for similar prompts` → `Analyze results from store`
    - `Analyze results from store` → `Is this a cache hit?`
    - `Is this a cache hit?` true → `Respond to Chat (from semantic cache)`
    - `Is this a cache hit?` false → `LLM Agent`
    - `OpenAI Chat Model` → `LLM Agent` (AI language model)
    - `Redis Chat Memory` → `LLM Agent` (AI memory)
    - `LLM Agent` → `Recursive Character Text Splitter`
    - `Recursive Character Text Splitter` → `Add response as metadata`
    - `Add response as metadata` → `Embeddings HuggingFace Inference1`
    - `Embeddings HuggingFace Inference1` → `Store entry in cache`
    - `LLM Agent` → `Store entry in cache`
    - `Add response as metadata` → `Store entry in cache`
    - `Store entry in cache` → `Respond to Chat (from LLM)`

16. **Credentials Setup:**
    - OpenAI API key for `OpenAI Chat Model`.
    - HuggingFace API key for both `Embeddings HuggingFace Inference` nodes.
    - Redis credentials with access to Redis server 8.x+ for `Check for similar prompts`, `Redis Chat Memory`, and `Store entry in cache`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                               | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow implements an intelligent semantic caching system that reduces LLM costs and improves response times by caching similar queries. Cache hits return instantly, while misses trigger fresh LLM calls. Requires OpenAI, HuggingFace, and Redis setup. | Main sticky note content embedded in node `Sticky Note`                                                     |
| Adjust the `distanceThreshold` in the `Analyze results from store` node to control cache sensitivity: lower values increase precision but cause more LLM calls; higher values accept more cache hits but may reduce relevance.                              | Sticky note `Sticky Note2`                                                                                   |
| Using your own embedding model can increase performance and reduce costs compared to popular external models. Embeddings calls are cheaper than chat model calls, so semantic caching can drastically save cost.                                           | Sticky note `Sticky Note3`                                                                                   |
| Redis server version 8.x is required. For older Redis versions, the Redis Query Engine module must be installed to support vector search capabilities.                                                                                                   | Mentioned in main sticky note                                                                                |
| The LLM Agent uses GPT-4.1-mini model with conversation memory saved in Redis to provide contextualized and consistent responses.                                                                                                                         | OpenAI Chat Model and Redis Chat Memory node details                                                         |
| Cache entries are stored as vectors with associated metadata including the response (`reply`), enabling semantic similarity search and quick retrieval.                                                                                                  | See nodes `Add response as metadata` and `Store entry in cache`                                              |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects the applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.