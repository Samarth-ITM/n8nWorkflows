Build a Knowledge-Based WhatsApp Assistant with RAG, Gemini, Supabase & Google Docs

https://n8nworkflows.xyz/workflows/build-a-knowledge-based-whatsapp-assistant-with-rag--gemini--supabase---google-docs-8865


# Build a Knowledge-Based WhatsApp Assistant with RAG, Gemini, Supabase & Google Docs

### 1. Workflow Overview

This workflow implements a Knowledge-Based WhatsApp Assistant using Retrieval-Augmented Generation (RAG) powered by Google Gemini AI, Supabase vector database, and Google Docs content. It targets scenarios where users interact via WhatsApp, asking questions that are answered based on embedded documentation content, enabling contextually relevant and up-to-date responses.

The workflow logic is divided into the following blocks:

- **1.1 Input Reception:** Captures incoming WhatsApp messages to trigger the assistant.
- **1.2 Knowledge Base Preparation:** Loads Google Docs content, splits it into chunks, creates embeddings, and stores them in Supabase for retrieval.
- **1.3 User Query Processing:** Embeds the user’s WhatsApp message, searches for relevant knowledge in Supabase, aggregates results, and sends to the AI agent.
- **1.4 AI Response Generation:** Uses Google Gemini to generate answers based on retrieved knowledge and sends the response back via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for incoming WhatsApp messages to trigger the workflow and branches based on message presence.

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - If

- **Node Details:**

  1. **WhatsApp Trigger**  
     - Type: WhatsApp Trigger node  
     - Role: Captures incoming WhatsApp messages as webhook events.  
     - Configuration: Default webhook listening for any WhatsApp message. Executes once per message.  
     - Inputs: External WhatsApp messages via webhook.  
     - Outputs: Passes message data to the "If" node.  
     - Edge cases: Webhook failures or authentication issues with WhatsApp API.

  2. **If**  
     - Type: If node  
     - Role: Checks if the incoming WhatsApp message contains content to process.  
     - Configuration: Likely checks for non-empty message text (not explicit in JSON).  
     - Inputs: Receives data from WhatsApp Trigger.  
     - Outputs: If true, proceeds to "Embend User Message"; if false, ends flow.  
     - Edge cases: Empty or unsupported message types could cause workflow to halt here.

---

#### 1.2 Knowledge Base Preparation

- **Overview:**  
  Loads source documents from Google Docs, splits text into manageable chunks, creates embeddings for each chunk, and stores these embeddings in Supabase for later retrieval.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Content for the Training (Google Docs)  
  - Splitting into Chunks (Code)  
  - Embedding Uploaded document (HTTP Request)  
  - Save the embedding in DB (Supabase)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Allows manual initiation of the embedding preparation process.  
     - Configuration: No parameters.  
     - Inputs: None.  
     - Outputs: Starts the chain to load and embed documents.  
     - Edge cases: Workflow must be triggered manually to update knowledge base.

  2. **Content for the Training**  
     - Type: Google Docs node  
     - Role: Fetches the source document content used for knowledge base creation.  
     - Configuration: Likely configured with Google Docs credentials and document ID (not visible in JSON).  
     - Inputs: Triggered by manual start.  
     - Outputs: Google Docs content sent to splitting node.  
     - Edge cases: Credential expiry, document not found, or access denied errors.

  3. **Splitting into Chunks**  
     - Type: Code node (JavaScript)  
     - Role: Splits the Google Docs content into smaller text chunks suitable for embedding.  
     - Configuration: Custom code (not shown) that likely divides text by paragraphs or tokens.  
     - Inputs: Google Docs content.  
     - Outputs: Array of text chunks.  
     - Edge cases: Improper splitting may cause context loss or overlap.

  4. **Embedding Uploaded document**  
     - Type: HTTP Request node  
     - Role: Sends each chunk to an embedding API (likely OpenAI or Google embedding endpoint) to generate vector representations.  
     - Configuration: HTTP POST with chunk text, API key in headers.  
     - Inputs: Text chunks from the code node.  
     - Outputs: Embeddings per chunk.  
     - Edge cases: API rate limits, key invalidation, or malformed requests.

  5. **Save the embedding in DB**  
     - Type: Supabase node  
     - Role: Stores embeddings and associated metadata in a Supabase vector database.  
     - Configuration: Supabase credentials and table defined; inserts embedding vectors.  
     - Inputs: Embeddings from HTTP Request.  
     - Outputs: Confirmation of storage.  
     - Edge cases: DB connection failures, schema mismatches, or data conflicts.

---

#### 1.3 User Query Processing

- **Overview:**  
  Processes the user's message by embedding it, searching the Supabase DB for relevant chunks, and aggregating the results for the AI agent.

- **Nodes Involved:**  
  - Embend User Message (HTTP Request)  
  - Search Embeddings (HTTP Request)  
  - Aggregate (Aggregate)

- **Node Details:**

  1. **Embend User Message**  
     - Type: HTTP Request node  
     - Role: Sends the user’s WhatsApp message text to the embedding API to create a vector.  
     - Configuration: HTTP POST with user message, API key in headers.  
     - Inputs: User message from "If" node.  
     - Outputs: Embedding vector for the message.  
     - Edge cases: Same as document embedding (rate limit, auth failure).

  2. **Search Embeddings**  
     - Type: HTTP Request node  
     - Role: Queries Supabase vector DB to find document chunks most relevant to the user message embedding.  
     - Configuration: HTTP POST/GET querying Supabase with embedding vector and similarity search parameters.  
     - Inputs: Embedding of user message.  
     - Outputs: List of top relevant document chunks.  
     - Edge cases: Query failures, empty search results.

  3. **Aggregate**  
     - Type: Aggregate node  
     - Role: Combines multiple relevant document chunks into a single data structure to provide context for AI.  
     - Configuration: Custom aggregation (e.g., concatenating text fields).  
     - Inputs: Search results.  
     - Outputs: Aggregated document context for AI agent.  
     - Edge cases: Overly long aggregate may hit token limits downstream.

---

#### 1.4 AI Response Generation and Delivery

- **Overview:**  
  Uses Google Gemini chat model to generate a response based on aggregated context and user query, then sends the response back via WhatsApp.

- **Nodes Involved:**  
  - Google Gemini Chat Model (LM Chat Google Gemini)  
  - AI Agent (Langchain Agent)  
  - Send message (WhatsApp)

- **Node Details:**

  1. **Google Gemini Chat Model**  
     - Type: Langchain Google Gemini Chat Model node  
     - Role: Provides AI language model capabilities using Google Gemini.  
     - Configuration: Credentials for Google Gemini API, model options default.  
     - Inputs: Receives aggregated document context and user query from "AI Agent".  
     - Outputs: AI-generated answer.  
     - Edge cases: API limits, response timeouts, malformed input.

  2. **AI Agent**  
     - Type: Langchain Agent  
     - Role: Orchestrates AI model with context and user messages; manages prompt construction and response extraction.  
     - Configuration: Utilizes Google Gemini as language model.  
     - Inputs: Aggregated context from "Aggregate" and AI model from "Google Gemini Chat Model".  
     - Outputs: Final AI response text.  
     - Edge cases: Prompt construction failures, empty responses.

  3. **Send message**  
     - Type: WhatsApp node  
     - Role: Sends AI-generated response back to the user on WhatsApp.  
     - Configuration: Configured with WhatsApp API webhook and credentials.  
     - Inputs: AI Agent’s output message.  
     - Outputs: Message delivery confirmation.  
     - Edge cases: WhatsApp API downtime, message formatting errors.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                        |
|---------------------------|----------------------------------|---------------------------------------|--------------------------|-------------------------|----------------------------------|
| WhatsApp Trigger          | WhatsApp Trigger                 | Receives incoming WhatsApp messages   | (External webhook)       | If                      |                                  |
| If                        | If                              | Checks if message content exists      | WhatsApp Trigger          | Embend User Message      |                                  |
| Embend User Message        | HTTP Request                    | Embeds user WhatsApp message           | If                       | Search Embeddings        |                                  |
| Search Embeddings          | HTTP Request                    | Searches Supabase for relevant docs   | Embend User Message       | Aggregate                |                                  |
| Aggregate                 | Aggregate                       | Combines relevant document chunks     | Search Embeddings         | AI Agent                 |                                  |
| AI Agent                  | Langchain Agent                 | Manages AI prompt and response flow   | Aggregate, Google Gemini Chat Model | Send message         |                                  |
| Google Gemini Chat Model   | LM Chat Google Gemini           | Generates AI language model responses | AI Agent (ai_languageModel) | AI Agent                |                                  |
| Send message              | WhatsApp                        | Sends response back to WhatsApp user  | AI Agent                  |                         |                                  |
| When clicking ‘Execute workflow’| Manual Trigger              | Manual start of knowledge base update |                          | Content for the Training |                                  |
| Content for the Training   | Google Docs                    | Retrieves Google Docs content          | When clicking ‘Execute workflow’ | Splitting into Chunks |                                  |
| Splitting into Chunks      | Code                           | Splits document into chunks            | Content for the Training  | Embedding Uploaded document |                                  |
| Embedding Uploaded document| HTTP Request                   | Embeds document chunks                 | Splitting into Chunks     | Save the embedding in DB |                                  |
| Save the embedding in DB   | Supabase                       | Stores embeddings in Supabase DB       | Embedding Uploaded document |                         |                                  |
| Sticky Note               | Sticky Note                    | (No content)                           |                          |                         |                                  |
| Sticky Note1              | Sticky Note                    | (No content)                           |                          |                         |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Node type: WhatsApp Trigger  
   - Configure webhook for incoming WhatsApp messages.  
   - Set to execute once per message.

2. **Add If node**  
   - Node type: If  
   - Condition: Check if incoming message text is not empty.  
   - Connect WhatsApp Trigger output to this node.

3. **Add HTTP Request node "Embend User Message"**  
   - Node type: HTTP Request  
   - Configure to call embedding API (e.g., OpenAI or Google embedding API).  
   - Pass user message text as request body.  
   - Connect If node’s true output here.

4. **Add HTTP Request node "Search Embeddings"**  
   - Node type: HTTP Request  
   - Configure to query Supabase vector DB with embedding vector from previous node.  
   - Use similarity search parameters.  
   - Connect Embend User Message output here.

5. **Add Aggregate node**  
   - Node type: Aggregate  
   - Configure to combine retrieved document chunks into a single string or array.  
   - Connect Search Embeddings output here.

6. **Add Langchain Google Gemini Chat Model node**  
   - Node type: Google Gemini Chat Model  
   - Configure with Google API credentials.  
   - No additional parameters needed.  

7. **Add Langchain Agent node**  
   - Node type: Langchain Agent  
   - Configure to use Google Gemini Chat Model as language model.  
   - Connect Aggregate output to agent input.  
   - Connect Google Gemini Chat Model output as AI language model input to agent.

8. **Add WhatsApp node "Send message"**  
   - Node type: WhatsApp  
   - Configure WhatsApp API credentials.  
   - Connect AI Agent output here to send AI response back.

9. **Create Manual Trigger node "When clicking ‘Execute workflow’"**  
   - Node type: Manual Trigger  
   - This node initiates the knowledge base update flow.

10. **Add Google Docs node "Content for the Training"**  
    - Node type: Google Docs  
    - Configure with Google credentials and specify target training document.  
    - Connect Manual Trigger output here.

11. **Add Code node "Splitting into Chunks"**  
    - Node type: Code  
    - Implement JavaScript code to split document text into chunks suitable for embedding (e.g., paragraphs or token-based).  
    - Connect Google Docs output here.

12. **Add HTTP Request node "Embedding Uploaded document"**  
    - Node type: HTTP Request  
    - Configure to call embedding API to embed each chunk.  
    - Connect Code node output here.

13. **Add Supabase node "Save the embedding in DB"**  
    - Node type: Supabase  
    - Configure Supabase credentials and specify table for embeddings.  
    - Connect HTTP Request node output here.

14. **Connect all nodes accordingly:**  
    - WhatsApp Trigger → If → Embend User Message → Search Embeddings → Aggregate → AI Agent → Send message  
    - Google Gemini Chat Model → AI Agent (ai_languageModel input)  
    - Manual Trigger → Content for the Training → Splitting into Chunks → Embedding Uploaded document → Save the embedding in DB

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                                                                                 |
|------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini via Langchain integration for advanced conversational AI.       | n8n Langchain Integration documentation: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                                        |
| Supabase is used as a vector database to enable fast similarity search on embedded document chunks. | Supabase Vector Search Docs: https://supabase.com/docs/guides/database/vector-search                                                                              |
| WhatsApp Trigger and WhatsApp nodes require appropriate WhatsApp Business API setup and credentials. | WhatsApp Business API documentation: https://developers.facebook.com/docs/whatsapp/                                                                              |
| The “Splitting into Chunks” node uses custom JavaScript; consider token limits for embeddings.   | Recommended chunk size is 500 tokens or less to optimize embedding quality and AI prompt length.                                                                |
| Manual Trigger node exists to update the knowledge base when Google Docs content changes.        | Run this periodically or on-demand after updating source documents to keep knowledge base current.                                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.