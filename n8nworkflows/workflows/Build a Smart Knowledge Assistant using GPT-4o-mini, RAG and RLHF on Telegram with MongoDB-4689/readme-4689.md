Build a Smart Knowledge Assistant using GPT-4o-mini, RAG and RLHF on Telegram with MongoDB

https://n8nworkflows.xyz/workflows/build-a-smart-knowledge-assistant-using-gpt-4o-mini--rag-and-rlhf-on-telegram-with-mongodb-4689


# Build a Smart Knowledge Assistant using GPT-4o-mini, RAG and RLHF on Telegram with MongoDB

### 1. Workflow Overview

This workflow implements a **Smart Knowledge Assistant** on Telegram, leveraging GPT-4o-mini, Retrieval-Augmented Generation (RAG), and Reinforcement Learning with Human Feedback (RLHF) integrated with MongoDB. Its main purpose is to provide internal users with accurate, context-aware answers by consulting indexed product documentation and past user feedback stored in a vector-based MongoDB store. The workflow supports continuous learning by collecting user feedback on responses, enhancing future replies.

The workflow is logically organized into the following blocks:

- **1.1 Documentation Import and Indexing**: Imports Google Docs content, splits it into chunks, generates embeddings, and stores them in MongoDB for fast semantic search.
- **1.2 Telegram Message Reception & Processing**: Listens for user questions on Telegram, manages chat memory, and triggers the AI agent.
- **1.3 AI Agent Query & Response Generation**: Uses GPT-4o-mini and LangChain to query the vector store (documentation + positive/negative feedback), generate answers, and apply behavior rules.
- **1.4 Telegram Response and Feedback Collection**: Sends answers back to users on Telegram, waits for feedback (positive/negative), maps feedback data, and stores it with embeddings in MongoDB for RLHF.

---

### 2. Block-by-Block Analysis

#### 2.1 Documentation Import and Indexing

**Overview:**  
This block imports official product documentation from Google Docs, processes the content by chunking and embedding generation, and stores the results in MongoDB with vector indexing for efficient retrieval.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Google Docs Importer  
- Document Section Loader  
- Document Chunker  
- OpenAI Embeddings Generator  
- MongoDB Documentation Inserter  

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Entry point to start the documentation import and indexing process manually.  
  - Connections: Starts Google Docs Importer.  
  - Edge cases: Manual trigger required; no automatic scheduling.

- **Google Docs Importer**  
  - Type: Google Docs node  
  - Role: Retrieves content from a specified Google Docs URL.  
  - Configuration: Operation "get", static document URL defined.  
  - Credentials: Google Docs OAuth2.  
  - Output: Raw document content.  
  - Edge cases: OAuth failure, network errors, document access permissions.

- **Document Section Loader**  
  - Type: Document Default Data Loader (LangChain)  
  - Role: Prepares document content for processing, adds metadata (doc_id).  
  - Configuration: Loads JSON content expression from Google Docs output; attaches document ID metadata.  
  - Input: Google Docs Importer output.  
  - Output: Document content ready for chunking.  
  - Edge cases: Invalid JSON or missing documentId fields.

- **Document Chunker**  
  - Type: Recursive Character Text Splitter  
  - Role: Splits loaded document into chunks of 3000 characters with 200 overlap; uses markdown splitCode.  
  - Input: Document Section Loader output.  
  - Output: Array of text chunks for embedding.  
  - Edge cases: Very large documents may generate many chunks; chunk size/overlap tuning needed.

- **OpenAI Embeddings Generator**  
  - Type: OpenAI Embeddings Node  
  - Role: Generates 1536-dimensional vector embeddings for each text chunk using OpenAI API.  
  - Credentials: OpenAI API key configured.  
  - Input: Document chunks.  
  - Output: Embedding vectors matched to chunks.  
  - Edge cases: API rate limits, invalid API key, network errors.

- **MongoDB Documentation Inserter**  
  - Type: MongoDB Atlas Vector Store Inserter  
  - Role: Inserts embedded chunks into MongoDB collection "n8n-template" with vector index "data_index".  
  - Configuration: Insert mode, no additional options.  
  - Credentials: MongoDB account configured.  
  - Input: Embeddings with metadata.  
  - Output: Confirmation of stored vectors.  
  - Edge cases: MongoDB connection errors, index misconfiguration.

---

#### 2.2 Telegram Message Reception & Processing

**Overview:**  
This block listens for incoming Telegram messages, extracts chat context, and manages chat memory in MongoDB to maintain conversational state across messages.

**Nodes Involved:**  
- Receive Message on Telegram  
- MongoDB Chat Memory  

**Node Details:**

- **Receive Message on Telegram**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (update type "message").  
  - Configuration: Webhook ID provided for secure incoming messages.  
  - Credentials: Telegram API OAuth token.  
  - Output: Incoming message JSON including chat ID and text.  
  - Edge cases: Telegram webhook misconfiguration, network downtime.

- **MongoDB Chat Memory**  
  - Type: LangChain MongoDB Chat Memory Node  
  - Role: Retrieves and stores chat history keyed by Telegram chat ID to maintain conversation context.  
  - Configuration: Session key set to Telegram chat ID, collection "n8n-template-chat-history".  
  - Credentials: MongoDB account.  
  - Input: Telegram message chat ID from previous node.  
  - Output: Chat memory context for AI agent.  
  - Edge cases: MongoDB unavailability, inconsistent session keys.

---

#### 2.3 AI Agent Query & Response Generation

**Overview:**  
This block integrates multiple tools for retrieval-augmented generation: it searches the MongoDB vector stores for relevant product documentation and user feedback (positive and negative), then uses GPT-4o-mini to generate a tailored response according to strict behavior rules.

**Nodes Involved:**  
- Knowledge Base Agent  
- Search Documentation  
- Search Positive Interactions  
- Search Negative Interactions  
- OpenAI Chat Model  

**Node Details:**

- **Knowledge Base Agent**  
  - Type: LangChain Agent Node  
  - Role: Central AI agent that receives user question text and uses multiple retrieval tools to generate an answer.  
  - Configuration:  
    - Takes the incoming message text as prompt.  
    - System message defines behavior rules: consult productDocs first, check positive and negative feedback, no links, language detection per message, concise professional responses.  
    - Tools enabled: productDocs (documentation), feedbackPositive, feedbackNegative.  
  - Input: Message text from Telegram + chat memory + tool outputs.  
  - Output: Generated AI response text.  
  - Version: v1.9 LangChain agent node.  
  - Edge cases: Tool search failures, missing relevant data, model API rate limit or failures.

- **Search Documentation**  
  - Type: MongoDB Atlas Vector Store Retrieval (LangChain)  
  - Role: Searches "n8n-template" MongoDB collection for relevant product documentation embeddings.  
  - Configuration:  
    - Mode: retrieve-as-tool  
    - Tool name: productDocs  
    - Vector index: data_index  
  - Credentials: MongoDB account.  
  - Input: Embeddings generated from the message text.  
  - Output: Relevant document snippets.  
  - Edge cases: MongoDB query failures, empty search results.

- **Search Positive Interactions**  
  - Type: MongoDB Atlas Vector Store Retrieval (LangChain)  
  - Role: Searches "n8n-template-feedback" collection for positively rated previous interactions.  
  - Configuration:  
    - Mode: retrieve-as-tool  
    - Tool name: feedbackPositive  
    - Metadata filter: feedback = positive  
    - Vector index: template-feedback-search  
  - Credentials: MongoDB account.  
  - Input: Message embeddings.  
  - Output: Examples of positively rated responses.  
  - Edge cases: No positive samples found, DB errors.

- **Search Negative Interactions**  
  - Type: MongoDB Atlas Vector Store Retrieval (LangChain)  
  - Role: Searches "n8n-template-feedback" collection for negatively rated previous responses to avoid repeating mistakes.  
  - Configuration:  
    - Mode: retrieve-as-tool  
    - Tool name: feedbackNegative  
    - Metadata filter: feedback = negative  
    - Vector index: template-feedback-search  
  - Credentials: MongoDB account.  
  - Input: Message embeddings.  
  - Output: Negative examples to avoid.  
  - Edge cases: No negative feedback found, DB failures.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model Node  
  - Role: Runs GPT-4o-mini model for language generation based on aggregated inputs from the agent.  
  - Configuration: Model set to "gpt-4o-mini" with RLHF enabled (reinforcement learning from human feedback).  
  - Credentials: OpenAI API key.  
  - Edge cases: API quota exceeded, model unavailability, latency.

---

#### 2.4 Telegram Response and Feedback Collection

**Overview:**  
This block sends the AI-generated answer back on Telegram, waits for user feedback (approval/rejection), processes and maps feedback data, generates embeddings for feedback text, and stores it in MongoDB for continuous learning.

**Nodes Involved:**  
- Send Message on Telegram, Wait for Feedback  
- Map feedback data  
- Set feedback fields for collection storage  
- OpenAI Embeddings Generator1  
- Default Data Loader  
- Recursive Character Text Splitter  
- Submit embedded chat feedback  

**Node Details:**

- **Send Message on Telegram, Wait for Feedback**  
  - Type: Telegram node (sendAndWait)  
  - Role: Sends the AI response message to the user and waits for double approval feedback (positive or negative).  
  - Configuration:  
    - Chat ID from incoming Telegram message.  
    - Sends the response text from Knowledge Base Agent.  
    - Approval type: double (positive/negative).  
  - Credentials: Telegram API.  
  - Edge cases: Telegram downtime, user does not respond with feedback.

- **Map feedback data**  
  - Type: Code Node (JavaScript)  
  - Role: Maps Telegram feedback boolean to "positive" or "negative" string for storage.  
  - Code logic: If approved is true, feedback = "positive", else "negative".  
  - Edge cases: Missing or malformed feedback data.

- **Set feedback fields for collection storage**  
  - Type: Set Node  
  - Role: Prepares structured data for feedback insertion into MongoDB, including prompt, response, full text, and feedback label.  
  - Values:  
    - prompt: original user message text  
    - response: AI generated answer  
    - text: concatenated prompt and response string  
    - feedback: mapped feedback ("positive" or "negative")  
  - Edge cases: Missing input data if upstream nodes fail.

- **OpenAI Embeddings Generator1**  
  - Type: OpenAI Embeddings Node  
  - Role: Generates vector embeddings of the feedback text for semantic storage.  
  - Credentials: OpenAI API.  
  - Edge cases: API failures, rate limiting.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter (for feedback text)  
  - Role: Splits feedback text into manageable chunks (4000 characters) for embedding and storing.  
  - Edge cases: Very long feedback texts.

- **Default Data Loader**  
  - Type: Document Default Data Loader  
  - Role: Loads feedback text chunks with metadata (prompt, response, feedback) for insertion.  
  - Edge cases: Misalignment in metadata fields.

- **Submit embedded chat feedback**  
  - Type: MongoDB Atlas Vector Store Inserter  
  - Role: Inserts feedback text embeddings and metadata into "n8n-template-feedback" collection with vector index "template-feedback-search".  
  - Credentials: MongoDB account.  
  - Edge cases: MongoDB insertion errors, index misconfiguration.

---

### 3. Summary Table

| Node Name                          | Node Type                                     | Functional Role                                    | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                                         |
|----------------------------------|-----------------------------------------------|---------------------------------------------------|--------------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                                | Manual start for documentation import             |                                | Google Docs Importer               | Run this workflow manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search. |
| Google Docs Importer              | Google Docs node                              | Fetches Google Docs content                        | When clicking "Execute Workflow" | MongoDB Documentation Inserter     |                                                                                                                                    |
| Document Section Loader           | Document Default Data Loader (LangChain)     | Prepares document content with metadata           | Document Chunker                | MongoDB Documentation Inserter     |                                                                                                                                    |
| Document Chunker                 | Recursive Character Text Splitter             | Splits docs into chunks                            | Document Section Loader          | OpenAI Embeddings Generator        |                                                                                                                                    |
| OpenAI Embeddings Generator       | OpenAI Embeddings                             | Generates vector embeddings for doc chunks        | Document Chunker                | MongoDB Documentation Inserter     |                                                                                                                                    |
| MongoDB Documentation Inserter    | MongoDB Vector Store Inserter                 | Stores embeddings and docs in MongoDB              | Google Docs Importer, Embeddings OpenAI |                                |                                                                                                                                    |
| Receive Message on Telegram       | Telegram Trigger                              | Receives Telegram user messages                    |                                | Knowledge Base Agent               |                                                                                                                                    |
| MongoDB Chat Memory               | LangChain MongoDB Chat Memory                 | Maintains conversation history                     | Receive Message on Telegram     | Knowledge Base Agent               |                                                                                                                                    |
| Knowledge Base Agent              | LangChain Agent Node                          | Main AI agent processing queries                   | Receive Message on Telegram, Search Documentation, Search Positive Interactions, Search Negative Interactions, MongoDB Chat Memory | Send Message on Telegram, Wait for Feedback | This workflow uses retrieval-augmented generation (RAG) to answer user questions by searching the MongoDB vector store and generating AI responses enhanced with Telegram-based RLHF feedback. |
| Search Documentation             | MongoDB Vector Store Retrieval (LangChain)   | Search product documentation embeddings            | Embeddings OpenAI               | Knowledge Base Agent               |                                                                                                                                    |
| Search Positive Interactions      | MongoDB Vector Store Retrieval (LangChain)   | Search positive feedback examples                   | Embeddings OpenAI1              | Knowledge Base Agent               |                                                                                                                                    |
| Search Negative Interactions      | MongoDB Vector Store Retrieval (LangChain)   | Search negative feedback examples                   | Embeddings OpenAI3              | Knowledge Base Agent               |                                                                                                                                    |
| OpenAI Chat Model                 | LangChain OpenAI Chat Model                   | Runs GPT-4o-mini model for answer generation       | Knowledge Base Agent            | Knowledge Base Agent               |                                                                                                                                    |
| Send Message on Telegram, Wait for Feedback | Telegram SendAndWait                     | Sends answer and waits for user feedback           | Knowledge Base Agent            | Map feedback data                 |                                                                                                                                    |
| Map feedback data                | Code Node                                     | Maps Telegram feedback to positive/negative labels | Send Message on Telegram, Wait for Feedback | Set feedback fields for collection storage |                                                                                                                                    |
| Set feedback fields for collection storage | Set Node                                  | Prepares feedback data for DB insertion             | Map feedback data               | Submit embedded chat feedback      |                                                                                                                                    |
| OpenAI Embeddings Generator1      | OpenAI Embeddings                             | Generates embeddings for feedback text             | Set feedback fields for collection storage | Submit embedded chat feedback      |                                                                                                                                    |
| Recursive Character Text Splitter | Text Splitter                                | Splits feedback text for embedding                  | Set feedback fields for collection storage | Default Data Loader                |                                                                                                                                    |
| Default Data Loader               | Document Default Data Loader (LangChain)     | Loads feedback text and metadata                     | Recursive Character Text Splitter | Submit embedded chat feedback      |                                                                                                                                    |
| Submit embedded chat feedback     | MongoDB Vector Store Inserter                  | Inserts feedback embeddings and metadata into MongoDB | Default Data Loader             |                                  |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger:**  
   - Add a Manual Trigger node named "When clicking \"Execute Workflow\"". No parameters.

2. **Google Docs Importer:**  
   - Add Google Docs node named "Google Docs Importer".  
   - Operation: "get".  
   - Document URL: Set to the product documentation Google Docs URL.  
   - Credentials: Configure Google Docs OAuth2 credentials.  
   - Connect Manual Trigger → Google Docs Importer.

3. **Document Section Loader:**  
   - Add LangChain Document Default Data Loader node named "Document Section Loader".  
   - JSON Data: Set expression to load content from Google Docs output (`{{$json.content}}`).  
   - Metadata: Add "doc_id" from Google Docs document id (`{{$json.documentId}}`).  
   - Connect Google Docs Importer → Document Section Loader.

4. **Document Chunker:**  
   - Add Recursive Character Text Splitter node named "Document Chunker".  
   - Chunk size: 3000 characters; Overlap: 200.  
   - Split code: "markdown".  
   - Connect Document Section Loader → Document Chunker.

5. **OpenAI Embeddings Generator:**  
   - Add OpenAI Embeddings node named "OpenAI Embeddings Generator".  
   - Credentials: Add OpenAI API credentials.  
   - Connect Document Chunker → OpenAI Embeddings Generator.

6. **MongoDB Documentation Inserter:**  
   - Add MongoDB Atlas Vector Store Inserter node named "MongoDB Documentation Inserter".  
   - Mode: Insert.  
   - Mongo Collection: Set to the collection for docs (e.g., "n8n-template").  
   - Vector Index Name: "data_index".  
   - Credentials: Configure MongoDB credentials.  
   - Connect Google Docs Importer main and OpenAI Embeddings Generator (ai_embedding) → MongoDB Documentation Inserter.

7. **Telegram Trigger:**  
   - Add Telegram Trigger node named "Receive Message on Telegram".  
   - Updates: Select "message".  
   - Credentials: Configure Telegram bot API credentials.  
   - This is the main entry for user queries.

8. **MongoDB Chat Memory:**  
   - Add LangChain MongoDB Chat Memory node named "MongoDB Chat Memory".  
   - Session Key: Set to Telegram Chat ID (`{{$json.message.chat.id}}`).  
   - Collection Name: e.g., "n8n-template-chat-history".  
   - Credentials: MongoDB credentials.  
   - Connect Receive Message on Telegram → MongoDB Chat Memory.

9. **Search Documentation:**  
   - Add MongoDB Vector Store Retrieval node named "Search Documentation".  
   - Mode: retrieve-as-tool.  
   - Tool name: "productDocs".  
   - Mongo Collection: "n8n-template".  
   - Vector Index Name: "data_index".  
   - Credentials: MongoDB.  
   - Connect OpenAI Embeddings Generator (ai_embedding) → Search Documentation.

10. **Search Positive Interactions:**  
    - Add MongoDB Vector Store Retrieval node named "Search Positive Interactions".  
    - Mode: retrieve-as-tool.  
    - Tool name: "feedbackPositive".  
    - Metadata filter: feedback = positive.  
    - Mongo Collection: "n8n-template-feedback".  
    - Vector Index Name: "template-feedback-search".  
    - Credentials: MongoDB.  
    - Connect OpenAI Embeddings Generator1 (ai_embedding) → Search Positive Interactions.

11. **Search Negative Interactions:**  
    - Add MongoDB Vector Store Retrieval node named "Search Negative Interactions".  
    - Mode: retrieve-as-tool.  
    - Tool name: "feedbackNegative".  
    - Metadata filter: feedback = negative.  
    - Mongo Collection: "n8n-template-feedback".  
    - Vector Index Name: "template-feedback-search".  
    - Credentials: MongoDB.  
    - Connect OpenAI Embeddings Generator3 (ai_embedding) → Search Negative Interactions.

12. **Knowledge Base Agent:**  
    - Add LangChain Agent node named "Knowledge Base Agent".  
    - Parameters:  
      - Text: `{{$json.message.text}}` (from Telegram).  
      - System Message: Paste the detailed system message specifying behavior rules and tool usage (see section 2.3).  
      - Prompt Type: define.  
      - Tools: productDocs, feedbackPositive, feedbackNegative.  
    - Connect Receive Message on Telegram (main), Search Documentation, Search Positive Interactions, Search Negative Interactions, and MongoDB Chat Memory (ai_memory) → Knowledge Base Agent.

13. **OpenAI Chat Model:**  
    - Add LangChain OpenAI Chat Model node named "OpenAI Chat Model".  
    - Model: "gpt-4o-mini" with RLHF enabled.  
    - Credentials: OpenAI API.  
    - Connect Knowledge Base Agent (ai_languageModel) → OpenAI Chat Model.  
    - Connect OpenAI Chat Model (ai_languageModel) → Knowledge Base Agent (ai_languageModel).

14. **Send Message on Telegram, Wait for Feedback:**  
    - Add Telegram node named "Send Message on Telegram, Wait for Feedback".  
    - Chat ID: `{{$json.message.chat.id}}`.  
    - Message: `{{$json.output}}` (AI response).  
    - Operation: sendAndWait.  
    - Approval Type: double (positive/negative).  
    - Credentials: Telegram API.  
    - Connect Knowledge Base Agent (main) → Send Message on Telegram, Wait for Feedback.

15. **Map feedback data:**  
    - Add Code node named "Map feedback data".  
    - JS Code:  
      ```javascript
      let response;
      if ($input.first().json.data.approved) {
        response = "positive";
      } else {
        response = "negative";
      }
      return { feedback: response };
      ```  
    - Connect Send Message on Telegram, Wait for Feedback → Map feedback data.

16. **Set feedback fields for collection storage:**  
    - Add Set node named "Set feedback fields for collection storage".  
    - Fields:  
      - prompt: `{{$('Receive Message on Telegram').item.json.message.text}}`  
      - response: `{{$('Knowledge Base Agent').item.json.output}}`  
      - text: `Prompt: {{$('Receive Message on Telegram').item.json.message.text}}. Completion: {{$('Knowledge Base Agent').item.json.output}}`  
      - feedback: `{{$json.feedback}}` (from Map feedback data)  
    - Connect Map feedback data → Set feedback fields for collection storage.

17. **OpenAI Embeddings Generator1:**  
    - Add OpenAI Embeddings node named "OpenAI Embeddings Generator1".  
    - Credentials: OpenAI API.  
    - Connect Set feedback fields for collection storage → OpenAI Embeddings Generator1.

18. **Recursive Character Text Splitter:**  
    - Add Recursive Character Text Splitter node named "Recursive Character Text Splitter".  
    - Chunk size: 4000.  
    - Connect Set feedback fields for collection storage → Recursive Character Text Splitter.

19. **Default Data Loader:**  
    - Add Document Default Data Loader node named "Default Data Loader".  
    - JSON Data: Load from input chunk text.  
    - Metadata: prompt, response, feedback fields from Set node.  
    - Connect Recursive Character Text Splitter → Default Data Loader.

20. **Submit embedded chat feedback:**  
    - Add MongoDB Atlas Vector Store Inserter node named "Submit embedded chat feedback".  
    - Mode: Insert.  
    - Mongo Collection: "n8n-template-feedback".  
    - Vector Index Name: "template-feedback-search".  
    - Credentials: MongoDB.  
    - Connect Default Data Loader, OpenAI Embeddings Generator1 → Submit embedded chat feedback.  
    - Connect Set feedback fields for collection storage → Submit embedded chat feedback.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                                                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses RAG (Retrieval-Augmented Generation) combined with RLHF (Reinforcement Learning from Human Feedback) to improve AI answers over time on Telegram.                                                                                              | Sticky Note1 on the canvas explains this concept.                                                                                                                                                                                                                                |
| The MongoDB vector search indexes are configured with 1536-dimensional embeddings and cosine similarity for semantic search.                                                                                                                                     | Sticky Note2 provides the JSON mapping structure for document and feedback indexes.                                                                                                                                                                                              |
| Users are explicitly instructed that the AI will not provide links under any circumstance, complying with content policies.                                                                                                                                       | Enforced in the Knowledge Base Agent system message.                                                                                                                                                                                                                            |
| The workflow is designed to be triggered manually to import and index Google Docs documentation, then runs continuously listening for Telegram messages.                                                                                                         | Sticky Note explains manual trigger use.                                                                                                                                                                                                                                         |
| Credentials required: OpenAI API key with access to GPT-4o-mini, Google Docs OAuth2, MongoDB Atlas account with vector search enabled, Telegram Bot API token.                                                                                                    | Setup credentials carefully and keep tokens secure.                                                                                                                                                                                                                              |
| For best performance, ensure MongoDB vector indexes are properly created as per the provided mapping JSON schemas.                                                                                                                                               | See Sticky Note2 for mapping examples.                                                                                                                                                                                                                                           |
| The workflow supports multi-language detection and response, replying in the exact language of the incoming message without fallback to history.                                                                                                                | Enforced by strict system message instructions in the LangChain agent configuration.                                                                                                                                                                                            |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow, complying strictly with content policies. It contains no illegal, offensive, or protected content. All manipulated data is legal and public.