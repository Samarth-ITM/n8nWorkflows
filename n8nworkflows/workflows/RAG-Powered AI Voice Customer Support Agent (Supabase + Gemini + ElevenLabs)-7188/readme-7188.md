RAG-Powered AI Voice Customer Support Agent (Supabase + Gemini + ElevenLabs)

https://n8nworkflows.xyz/workflows/rag-powered-ai-voice-customer-support-agent--supabase---gemini---elevenlabs--7188


# RAG-Powered AI Voice Customer Support Agent (Supabase + Gemini + ElevenLabs)

### 1. Workflow Overview

This workflow implements a Retrieval-Augmented Generation (RAG) powered AI voice customer support agent that integrates Supabase for vector storage, Google Vertex AI Gemini as the large language model (LLM) backend, and ElevenLabs for voice synthesis (though ElevenLabs nodes are not explicitly present in the provided nodes, the title suggests voice capabilities). The workflow receives user queries via webhook, generates semantic embeddings, performs similarity search over stored embeddings in Supabase, aggregates relevant documents, and generates AI responses using the Gemini chat model. The responses are then sent back to the webhook caller.

The workflow can be logically divided into two main functional blocks:

- **1.1 Data Preparation and Embedding Ingestion**: This block handles loading and preprocessing training content from Google Docs, chunking the text, generating embeddings, and saving them into Supabase for retrieval.

- **1.2 Query Handling and AI Response Generation**: This block handles incoming user queries via webhook, generates embeddings for the query, performs similarity search in Supabase, aggregates retrieved contexts, and generates responses using Google Vertex Gemini chat model before returning the answer.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Preparation and Embedding Ingestion

- **Overview:**  
  This block is responsible for loading training or reference documents from Google Docs, splitting the content into manageable chunks, generating embeddings for each chunk, and storing these embeddings in a Supabase database. It runs when manually triggered.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Content for the Training` (Google Docs)  
  - `Splitting into Chunks` (Code)  
  - `Embedding Uploaded document` (HTTP Request)  
  - `Save the embedding in DB` (Supabase)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for ingestion workflow  
    - Configuration: No special parameters; manual start  
    - Inputs: None  
    - Outputs: Triggers `Content for the Training` node  
    - Failure modes: None, manual trigger only  

  - **Content for the Training**  
    - Type: Google Docs  
    - Role: Retrieves document content for embedding  
    - Configuration: Uses Google Docs credentials; likely configured to fetch a specific document (document ID not visible)  
    - Inputs: Trigger from manual node  
    - Outputs: Passes document content to chunking node  
    - Failure modes: Google Docs API errors, auth failures  

  - **Splitting into Chunks**  
    - Type: Code (JavaScript)  
    - Role: Splits retrieved document text into smaller chunks suitable for embedding  
    - Configuration: Custom code logic (not shown) to chunk text, e.g., by paragraphs or token count  
    - Inputs: Document text from Google Docs node  
    - Outputs: Array of text chunks for embedding  
    - Failure modes: Code errors, empty input text  

  - **Embedding Uploaded document**  
    - Type: HTTP Request  
    - Role: Calls embedding API to generate vector embeddings for chunks  
    - Configuration: Likely calls OpenAI or Gemini embedding endpoint with text chunks; configured with relevant credentials and headers  
    - Inputs: Text chunks from code node  
    - Outputs: Embeddings in JSON format  
    - Failure modes: API key expiration, rate limits, network timeouts, malformed requests  

  - **Save the embedding in DB**  
    - Type: Supabase  
    - Role: Stores generated embeddings with metadata into Supabase vector DB  
    - Configuration: Uses Supabase credentials, target table, and insert operation  
    - Inputs: Embeddings and chunk metadata from HTTP Request node  
    - Outputs: Confirmation of DB insert  
    - Failure modes: DB connection errors, insert conflicts, credential issues  

---

#### 2.2 Query Handling and AI Response Generation

- **Overview:**  
  This block handles live customer queries received via webhook, generates embeddings, searches for relevant documents in Supabase, aggregates retrieved content, queries the Gemini chat model for an AI-generated response, and returns the response to the requester.

- **Nodes Involved:**  
  - `Webhook`  
  - `Embend User Message` (HTTP Request)  
  - `Search Embeddings` (HTTP Request)  
  - `Aggregate`  
  - `Basic LLM Chain`  
  - `Google Vertex Chat Model`  
  - `Respond to Webhook`

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP POST endpoint)  
    - Role: Entry point for user queries  
    - Configuration: Exposes webhook URL for external calls; no additional parameters shown  
    - Inputs: External HTTP requests with user messages  
    - Outputs: Forwards raw user message to embedding generation node  
    - Failure modes: Network issues, invalid payloads, security concerns (no auth shown)  

  - **Embend User Message**  
    - Type: HTTP Request  
    - Role: Generates embedding for the user query message  
    - Configuration: Calls embedding API (likely Gemini or OpenAI), passing user text  
    - Inputs: User message from webhook  
    - Outputs: Embedding vector for query  
    - Failure modes: API failures, invalid user input, timeouts  

  - **Search Embeddings**  
    - Type: HTTP Request  
    - Role: Queries Supabase vector DB with user query embedding to find similar documents  
    - Configuration: HTTP call to Supabase REST or RPC endpoint for vector similarity search  
    - Inputs: Query embedding from previous node  
    - Outputs: List of relevant document chunks with similarity scores  
    - Failure modes: DB errors, no matches found, response format errors  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects and concatenates retrieved document chunks to form context  
    - Configuration: Default aggregation behavior (likely concatenation of texts)  
    - Inputs: Search results from Supabase  
    - Outputs: Single aggregated text block as context for AI model  
    - Failure modes: Empty input list, aggregation errors  

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Uses aggregated context and user query to generate AI response  
    - Configuration: Connects input context and prompt template to Google Vertex chat model  
    - Inputs: Aggregated context and user question  
    - Outputs: AI-generated text response  
    - Failure modes: Model API errors, prompt formatting errors  

  - **Google Vertex Chat Model**  
    - Type: LangChain Google Vertex AI Chat Model  
    - Role: Large language model backend generating conversation responses  
    - Configuration: Uses Google Cloud credentials to access Gemini chat model  
    - Inputs: Prompt and context from LLM chain  
    - Outputs: Text response returned to LLM Chain node  
    - Failure modes: Google Cloud auth issues, quota limits, malformed prompt  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends AI-generated response back to the original webhook caller  
    - Configuration: Default response node configuration  
    - Inputs: Response text from LLM chain  
    - Outputs: HTTP response to client  
    - Failure modes: Network errors, response formatting issues  

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                                  | Input Node(s)                        | Output Node(s)             | Sticky Note                          |
|---------------------------|------------------------------------|-------------------------------------------------|------------------------------------|----------------------------|------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                     | Starts document ingestion workflow              | -                                  | Content for the Training   |                                    |
| Content for the Training   | Google Docs                        | Retrieves training document content              | When clicking ‘Execute workflow’   | Splitting into Chunks      |                                    |
| Splitting into Chunks      | Code                              | Splits document text into chunks                  | Content for the Training           | Embedding Uploaded document|                                    |
| Embedding Uploaded document| HTTP Request                      | Generates embeddings for document chunks          | Splitting into Chunks              | Save the embedding in DB   |                                    |
| Save the embedding in DB   | Supabase                          | Saves embeddings into Supabase DB                  | Embedding Uploaded document        | -                          |                                    |
| Webhook                   | Webhook                          | Receives user queries                              | -                                  | Embend User Message        |                                    |
| Embend User Message        | HTTP Request                     | Generates embedding for user query                 | Webhook                           | Search Embeddings          |                                    |
| Search Embeddings          | HTTP Request                     | Searches Supabase for relevant documents           | Embend User Message               | Aggregate                  |                                    |
| Aggregate                 | Aggregate                        | Aggregates retrieved documents into context        | Search Embeddings                 | Basic LLM Chain            |                                    |
| Basic LLM Chain            | LangChain LLM Chain              | Generates AI response using context and query      | Aggregate                        | Respond to Webhook         |                                    |
| Google Vertex Chat Model   | LangChain Google Vertex AI Chat  | Provides LLM backend for generating text responses | Basic LLM Chain (ai_languageModel)| Basic LLM Chain            |                                    |
| Respond to Webhook         | Respond to Webhook               | Sends AI response back to the caller               | Basic LLM Chain                  | -                          |                                    |
| Sticky Note               | Sticky Note                      | (Empty content)                                    | -                                | -                          |                                    |
| Sticky Note1              | Sticky Note                      | (Empty content)                                    | -                                | -                          |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - No special parameters  

2. **Add Google Docs Node:**  
   - Type: Google Docs  
   - Name: "Content for the Training"  
   - Configure Google Docs credentials  
   - Set document ID to target training content document  
   - Connect output of Manual Trigger to this node  

3. **Add Code Node:**  
   - Type: Code (JavaScript)  
   - Name: "Splitting into Chunks"  
   - Configure code to split document text into chunks (e.g., by paragraphs or token counts)  
   - Input: document content from Google Docs node  
   - Output: array of text chunks  
   - Connect Google Docs to this node  

4. **Add HTTP Request Node for Embeddings:**  
   - Type: HTTP Request  
   - Name: "Embedding Uploaded document"  
   - Configure to call embedding API endpoint (OpenAI or Gemini)  
   - Use appropriate credentials and headers for API access  
   - Pass text chunks as request payload  
   - Connect Code node output to this node  

5. **Add Supabase Node:**  
   - Type: Supabase  
   - Name: "Save the embedding in DB"  
   - Configure Supabase credentials (URL, API key)  
   - Set insert operation to target embeddings table  
   - Map embedding vectors and chunk metadata to fields  
   - Connect HTTP Request node output to this node  

6. **Create Webhook Node:**  
   - Type: Webhook (POST)  
   - Name: "Webhook"  
   - Configure endpoint path and method  
   - No authentication shown but consider adding for production  
   - Entry point for live queries  

7. **Add HTTP Request Node for Query Embedding:**  
   - Type: HTTP Request  
   - Name: "Embend User Message"  
   - Configure to call embedding API with user query text from webhook  
   - Use credentials as per embedding service  
   - Connect Webhook output to this node  

8. **Add HTTP Request Node for Supabase Search:**  
   - Type: HTTP Request  
   - Name: "Search Embeddings"  
   - Configure to call Supabase REST or RPC endpoint for vector similarity search  
   - Pass query embedding vector from previous node  
   - Connect embedding node output to this node  

9. **Add Aggregate Node:**  
   - Type: Aggregate  
   - Name: "Aggregate"  
   - Configure aggregation to concatenate or compile retrieved document chunks into a single text context  
   - Connect Search Embeddings output to this node  

10. **Add LangChain Google Vertex Chat Model Node:**  
    - Type: LangChain Google Vertex AI Chat Model  
    - Name: "Google Vertex Chat Model"  
    - Configure with Google Cloud credentials and select Gemini model  
    - No direct input connection (used as AI language model for chain node)  

11. **Add LangChain LLM Chain Node:**  
    - Type: LangChain LLM Chain  
    - Name: "Basic LLM Chain"  
    - Configure with prompt template combining user query and aggregated context  
    - Use "Google Vertex Chat Model" node as AI language model  
    - Connect Aggregate node output to main input  
    - Connect AI model node to ai_languageModel input  

12. **Add Respond to Webhook Node:**  
    - Type: Respond to Webhook  
    - Name: "Respond to Webhook"  
    - Connect output of Basic LLM Chain to this node  
    - Sends AI response back to webhook caller  

13. **Connect all nodes accordingly:**  
    - Manual Trigger → Google Docs → Code → Embedding HTTP → Supabase save  
    - Webhook → User Embedding → Supabase Search → Aggregate → LLM Chain → Respond to Webhook  
    - Google Vertex Chat Model connected as ai_languageModel to LLM Chain node  

14. **Credentials Setup:**  
    - Google Docs OAuth2 credentials for document access  
    - Supabase credentials (URL, API key) for DB access  
    - OpenAI or Gemini credentials for embedding and chat models  
    - Google Cloud credentials for Vertex AI Gemini chat model  

15. **Validation:**  
    - Test embedding generation and DB insertion with manual trigger  
    - Test webhook with example user queries and verify AI response  
    - Monitor for errors such as API rate limits, invalid inputs, or DB failures  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow integrates Supabase vector DB, Google Vertex AI Gemini LLM, and is designed for voice support (ElevenLabs voice nodes implied). | Workflow title: RAG-Powered AI Voice Customer Support Agent (Supabase + Gemini + ElevenLabs) |
| For embedding and chat generation, ensure API keys and quotas are properly configured and monitored.      | Important for stable operation                     |
| Consider securing webhook endpoints with authentication for production environments.                      | Security best practices                            |
| Google Docs node requires OAuth2 credentials with scope for document read access.                          | Credential setup instructions                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.