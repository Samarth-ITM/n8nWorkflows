RAG Chatbot with Supabase + TogetherAI + Openrouter

https://n8nworkflows.xyz/workflows/rag-chatbot-with-supabase---togetherai---openrouter-5680


# RAG Chatbot with Supabase + TogetherAI + Openrouter

### 1. Workflow Overview

This n8n workflow implements a Retrieval-Augmented Generation (RAG) chatbot that integrates Supabase for vector database storage, TogetherAI for text embedding generation, and OpenRouter for AI language model responses. The design supports two main logical flows:

- **1.1 Content Preparation and Embedding Storage:**  
  This block fetches training content from a Google Doc, splits it into manageable text chunks, generates vector embeddings for each chunk using TogetherAI, and stores these embeddings along with their text chunks in a Supabase database table. This setup is intended to be run once to prepare the knowledge base for the chatbot.

- **1.2 User Query Handling and Response Generation:**  
  This block listens for user chat messages (via a LangChain chat trigger), embeds the incoming user message using TogetherAI, searches the Supabase vector database for the most similar context chunks, aggregates these chunks, and uses an LLM chain with OpenRouter’s `qwen/qwen3-8b:free` model to generate a context-aware answer. The LLM is explicitly instructed to answer based only on retrieved context, ensuring grounded responses.

The workflow currently has a disabled Telegram trigger as an alternative input mechanism but uses the LangChain chat trigger as the main entry point for user queries.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Preparation and Embedding Storage

- **Overview:**  
  This block prepares and stores the knowledge base embeddings by extracting content from Google Docs, chunking the text, generating embeddings for each chunk, and saving them in Supabase. It should be run only once or when updating the knowledge base.

- **Nodes Involved:**  
  - `Content for the Training` (Google Docs)  
  - `Splitting into Chunks` (Code)  
  - `Embedding Uploaded document` (HTTP Request)  
  - `Save the embedding in DB` (Supabase)  
  - `Telegram Trigger` (disabled)

- **Node Details:**

  - **Content for the Training**  
    - Type: Google Docs node  
    - Role: Fetches the full textual content of a specified Google Doc by URL using Service Account credentials.  
    - Configuration: Document URL specified by Google Docs document ID; authentication via Google Service Account.  
    - Inputs: Triggered by Telegram Trigger or manually run.  
    - Outputs: JSON with full document content as a single string under `content`.  
    - Failure cases: Google API auth errors, document not found, rate limits.

  - **Splitting into Chunks**  
    - Type: Code node (JavaScript)  
    - Role: Splits the large document text into 1000-character chunks to optimize embedding generation and storage.  
    - Key Logic: Slices the document string into multiple JSON items each containing a `chunk` field.  
    - Inputs: Receives document content JSON.  
    - Outputs: Multiple JSON items with `chunk` strings.  
    - Edge cases: Empty document content, chunk size exceeding limits, UTF-8 character boundary issues.

  - **Embedding Uploaded document**  
    - Type: HTTP Request node  
    - Role: Sends each chunk to TogetherAI embedding API to convert text into vector embeddings.  
    - Configuration: POST to `https://api.together.xyz/v1/embeddings`, sending JSON body with model `BAAI/bge-large-en-v1.5` and chunk text. Uses HTTP Bearer Auth with Together API key.  
    - Inputs: Each chunk item from previous node.  
    - Outputs: JSON with embedding vector under `data[0].embedding`.  
    - Failure modes: API auth failure, rate limit, connectivity issues, malformed requests.

  - **Save the embedding in DB**  
    - Type: Supabase node  
    - Role: Inserts each text chunk and its embedding vector into Supabase table `embed`.  
    - Configuration: Maps `chunk` text and `embedding` JSON string fields to Supabase table columns.  
    - Inputs: Embedding response JSONs.  
    - Outputs: Confirmation of DB insertions.  
    - Failure modes: DB auth failure, constraint violations, network errors.

  - **Telegram Trigger**  
    - Type: Telegram Trigger node (disabled)  
    - Role: Intended to listen for Telegram messages to initiate workflow (currently disabled).  
    - Inputs: External Telegram messages.  
    - Outputs: Triggers the content preparation flow if enabled.  
    - Failure modes: Telegram auth errors, webhook misconfiguration.

---

#### 2.2 User Query Handling and Response Generation

- **Overview:**  
  This block handles incoming chat messages, generates embeddings for the user query, retrieves relevant context chunks from Supabase, aggregates them, and uses an LLM chain to produce a context-based answer via OpenRouter.

- **Nodes Involved:**  
  - `When chat message received` (LangChain chatTrigger)  
  - `Embend User Message` (HTTP Request)  
  - `Search Embeddings` (HTTP Request)  
  - `Aggregate` (Aggregate node)  
  - `Basic LLM Chain` (LangChain LLM Chain)  
  - `OpenRouter Chat Model` (LangChain OpenRouter LLM)  

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Listens for user chat input to start the query processing flow.  
    - Configuration: Public webhook with initial greeting message.  
    - Inputs: Incoming chat messages with `chatInput` field.  
    - Outputs: User message JSON.  
    - Failure modes: Webhook downtime, malformed requests.

  - **Embend User Message**  
    - Type: HTTP Request node  
    - Role: Generates vector embedding for user’s chat input.  
    - Configuration: POST to TogetherAI embeddings API with model `BAAI/bge-large-en-v1.5` and user input text. Uses HTTP Bearer Auth.  
    - Inputs: User chatInput string.  
    - Outputs: Embedding vector JSON.  
    - Failure modes: API auth failure, rate limits, connectivity issues.

  - **Search Embeddings**  
    - Type: HTTP Request node  
    - Role: Queries Supabase RPC function `matchembeddings1` to find top 5 most similar text chunks based on user embedding.  
    - Configuration: POST to Supabase REST endpoint with parameters: user embedding vector and match_count=5. Uses Supabase API credentials.  
    - Inputs: User embedding vector JSON.  
    - Outputs: List of matched chunks with similarity scores.  
    - Failure modes: DB auth errors, RPC function issues, network errors.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Combines the matched text chunks into a single aggregated string to provide context for the LLM.  
    - Configuration: Aggregates on field `chunk` by concatenating or grouping.  
    - Inputs: List of chunk JSONs from Search Embeddings.  
    - Outputs: Single aggregated context string.  
    - Failure modes: Empty input sets, aggregation logic errors.

  - **Basic LLM Chain**  
    - Type: LangChain chainLlm node  
    - Role: Constructs a prompt with aggregated context and user question, directing the LLM to respond only based on context or admit lack of knowledge.  
    - Configuration: Prompt template combining context chunks and user input; batching enabled but no batch details specified.  
    - Inputs: Aggregated chunk string and user message.  
    - Outputs: LLM prompt JSON.  
    - Failure modes: Expression evaluation errors in prompt, missing inputs.

  - **OpenRouter Chat Model**  
    - Type: LangChain lmChatOpenRouter node  
    - Role: Provides the AI model backend (Qwen 3 8B free model) that generates the final answer from the prompt.  
    - Configuration: Model set to `qwen/qwen3-8b:free` with default options; authenticated via OpenRouter API credentials.  
    - Inputs: Prompt JSON from Basic LLM Chain.  
    - Outputs: AI-generated answer JSON.  
    - Failure modes: API auth failure, model unavailability, rate limits.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                  | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|-------------------------|--------------------------------|-------------------------------------------------|-----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger        | telegramTrigger                | Listens for Telegram messages to start workflow | None                        | Content for the Training   | *Currently disabled.* Intended to trigger content preparation block.                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Content for the Training | googleDocs                    | Fetches Google Docs content via Service Account | Telegram Trigger            | Splitting into Chunks      | Fetches document content to prepare embeddings for RAG knowledge base.                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Splitting into Chunks   | code                          | Splits document text into 1000-char chunks      | Content for the Training    | Embedding Uploaded document | Splits large text into manageable chunks for embedding.                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Embedding Uploaded document | httpRequest               | Calls TogetherAI embeddings API on each chunk   | Splitting into Chunks       | Save the embedding in DB   | Converts chunks into embeddings to store in DB.                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Save the embedding in DB | supabase                      | Saves chunk and embedding vector in Supabase DB | Embedding Uploaded document | None                      | Stores prepared knowledge base vectors into Supabase table `embed`.                                                                                                                                                                                                                                                                                                                                                                                                                      |
| When chat message received | chatTrigger (LangChain)      | Entry point for user chat queries                 | None                        | Embend User Message        | Starts user query flow with greeting.                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Embend User Message     | httpRequest                   | Embeds user chat input via TogetherAI            | When chat message received  | Search Embeddings          | Converts user query to embedding for similarity search.                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Search Embeddings       | httpRequest                   | Searches Supabase for top 5 similar chunks       | Embend User Message         | Aggregate                 | Finds relevant context chunks matching user query embedding.                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Aggregate               | aggregate                     | Aggregates matched text chunks into a single context | Search Embeddings           | Basic LLM Chain            | Combines retrieved chunks to provide context for LLM.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Basic LLM Chain         | chainLlm (LangChain)          | Constructs prompt and chains to LLM               | Aggregate                   | OpenRouter Chat Model      | Creates prompt instructing LLM to answer based only on context.                                                                                                                                                                                                                                                                                                                                                                                                                          |
| OpenRouter Chat Model   | lmChatOpenRouter (LangChain)  | Runs AI model to generate chatbot answer          | Basic LLM Chain             | None                      | Uses `qwen/qwen3-8b:free` model via OpenRouter to generate final response.                                                                                                                                                                                                                                                                                                                                                                                                               |
| Sticky Note             | stickyNote                    | Documentation note for Content Preparation block  | None                        | None                      | ⚠️ RUN THIS FIRST & RUN IT FOR ONLY ONCE: Converts content into embeddings and saves in DB, preparing knowledge base for RAG Chat. Explains included nodes in content preparation block.                                                                                                                                                                                                                                                                                                  |
| Sticky Note1            | stickyNote                    | Documentation note for User Query block            | None                        | None                      | Describes chat trigger, user embedding, search embeddings, aggregation, LLM chain, and OpenRouter Chat Model nodes in user query handling block.                                                                                                                                                                                                                                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Docs Node: "Content for the Training"**  
   - Type: `googleDocs`  
   - Operation: `get`  
   - Document URL: Enter Google Docs document ID (e.g., `1shUBSb2aEFZrOWOROcCeG9glxi7u205p4lEopNt1e7E`)  
   - Authentication: Use Google Service Account credentials  
   - Position on canvas: Left side (e.g., [-1880, -100])  

2. **Create Code Node: "Splitting into Chunks"**  
   - Type: `code`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const text = $input.first().json.content;
     const chunkSize = 1000;

     let chunks = [];
     for (let i = 0; i < text.length; i += chunkSize) {
       chunks.push({
         json: { chunk: text.slice(i, i + chunkSize) }
       });
     }
     return chunks;
     ```  
   - Connect output of "Content for the Training" to this node  
   - Position: Slightly right of previous node  

3. **Create HTTP Request Node: "Embedding Uploaded document"**  
   - Type: `httpRequest`  
   - Method: POST  
   - URL: `https://api.together.xyz/v1/embeddings`  
   - Authentication: HTTP Bearer Auth with Together API key (create credential)  
   - Headers: `Content-Type: application/json`  
   - Body Parameters (JSON):  
     ```json
     {
       "model": "BAAI/bge-large-en-v1.5",
       "input": "={{ $json.chunk }}"
     }
     ```  
   - Connect output of "Splitting into Chunks" to this node  
   - Position: Right of previous node  

4. **Create Supabase Node: "Save the embedding in DB"**  
   - Type: `supabase`  
   - Operation: Insert into table `embed`  
   - Field mappings:  
     - `chunk` → `={{ $('Splitting into Chunks').item.json.chunk }}`  
     - `embedding` → `={{ JSON.stringify($json.data[0].embedding) }}`  
   - Credentials: Supabase API with correct project and key  
   - Connect output of "Embedding Uploaded document" to this node  
   - Position: Right of previous node  

5. **(Optional) Create Telegram Trigger Node: "Telegram Trigger"**  
   - Type: `telegramTrigger`  
   - Select updates: `message`  
   - Credentials: Telegram Bot API token  
   - Disabled by default; used only if Telegram input desired  
   - Connect output to "Content for the Training" for initialization flow  

6. **Create Chat Trigger Node: "When chat message received"**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Public: true  
   - Initial Messages: `"Hi there! 👋\nTest the basic RAG chat with Supabase "`  
   - Position: Center-right of canvas  

7. **Create HTTP Request Node: "Embend User Message"**  
   - Type: `httpRequest`  
   - Method: POST  
   - URL: `https://api.together.xyz/v1/embeddings`  
   - Authentication: HTTP Bearer Auth with Together API key  
   - Headers: `Content-Type: application/json`  
   - Body Parameters:  
     ```json
     {
       "model": "BAAI/bge-large-en-v1.5",
       "input": "={{ $json.chatInput }}"
     }
     ```  
   - Connect output of "When chat message received" to this node  

8. **Create HTTP Request Node: "Search Embeddings"**  
   - Type: `httpRequest`  
   - Method: POST  
   - URL: `https://<your-supabase-project>.supabase.co/rest/v1/rpc/matchembeddings1` (replace domain)  
   - Authentication: Supabase API credentials  
   - Body Parameters:  
     ```json
     {
       "query_embedding": "={{ $json.data[0].embedding }}",
       "match_count": 5
     }
     ```  
   - Connect output of "Embend User Message" to this node  

9. **Create Aggregate Node: "Aggregate"**  
   - Type: `aggregate`  
   - Aggregate field: `chunk`  
   - Operation: Combine all chunks into one aggregated context string (default concatenation)  
   - Connect output of "Search Embeddings" to this node  

10. **Create LangChain Chain LLM Node: "Basic LLM Chain"**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Prompt Type: Define  
    - Text prompt:  
      ```
      You are an AI assistant. Use the following context to answer the user's question.

      Context:
      {{ $json.chunk }}

      User's message:
      {{ $('When chat message received').item.json.chatInput }}

      Provide a helpful and detailed answer based *only* on the context. If the answer is not in the context, say "I don't know based on the provided information."
      ```  
    - Connect output of "Aggregate" node to main input of this node  

11. **Create LangChain OpenRouter Chat Model Node: "OpenRouter Chat Model"**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Model: `qwen/qwen3-8b:free`  
    - Credentials: OpenRouter API key  
    - Connect output of this node to AI language model input of "Basic LLM Chain" node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow uses TogetherAI API for text embeddings: [TogetherAI](https://api.together.xyz)                                                                                                                                              | TogetherAI Embeddings API                                                                                 |
| Supabase is used as a vector database with a custom RPC function `matchembeddings1` to perform similarity search on embeddings. Ensure the RPC function exists and is properly configured in Supabase.                                   | Supabase Vector Search                                                                                     |
| The LangChain nodes require n8n version supporting `@n8n/n8n-nodes-langchain` package, version 1.7+ for `chainLlm` and 1+ for `lmChatOpenRouter`.                                                                                        | n8n LangChain Integration                                                                                  |
| The Telegram Trigger is disabled and can be enabled if Telegram bot integration is desired for input.                                                                                                                                     | Telegram Bot API                                                                                           |
| The prompt instructs the LLM to answer strictly based on retrieved context to avoid hallucinations, a best practice in RAG implementations.                                                                                              | Prompt Engineering Best Practices                                                                          |
| For embedding chunk size, 1000 characters is chosen to balance embedding quality and API limits; adjust as needed depending on text complexity and API token limits.                                                                        | Chunk Size Considerations                                                                                   |
| OpenRouter provides access to the Qwen model as a cost-free option; ensure API credentials are valid and rate limits are respected.                                                                                                      | OpenRouter API                                                                                             |
| The workflow is structured to separate knowledge base preparation (run once) and query answering (run per user request).                                                                                                                 | Workflow Design                                                                                             |

---

_Disclaimer: This document is derived entirely from an n8n workflow automation. It complies with all applicable content policies and contains no illegal, offensive, or protected materials. All processed data are lawful and public._