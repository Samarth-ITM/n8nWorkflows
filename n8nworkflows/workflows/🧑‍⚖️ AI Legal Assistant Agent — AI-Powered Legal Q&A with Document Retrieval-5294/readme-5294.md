🧑‍⚖️ AI Legal Assistant Agent — AI-Powered Legal Q&A with Document Retrieval

https://n8nworkflows.xyz/workflows/------ai-legal-assistant-agent---ai-powered-legal-q-a-with-document-retrieval-5294


# 🧑‍⚖️ AI Legal Assistant Agent — AI-Powered Legal Q&A with Document Retrieval

### 1. Workflow Overview

This workflow, titled **"🧑‍⚖️ AI Legal Assistant Agent — AI-Powered Legal Q&A with Document Retrieval"**, is designed to serve as an AI-powered legal assistant chatbot that answers users’ legal questions via Telegram. It uses a Retrieval-Augmented Generation (RAG) approach by combining OpenAI’s large language models with vector search capabilities provided by Pinecone, referencing a pre-indexed legal document library.

The workflow targets law firms, compliance teams, startups offering legal tech services, and anyone needing instant, accurate legal information from contracts, policies, or regulatory documents.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming messages from users through Telegram.
- **1.2 Contextual Memory Management:** Maintains conversation context using a memory buffer.
- **1.3 AI Processing:** Uses OpenAI language models and Pinecone vector search for retrieval-augmented generation (RAG) to answer legal queries accurately.
- **1.4 Output Delivery:** Sends the AI-generated legal answers back to the user on Telegram.
- **1.5 Document Embedding & Vector Store Management (background setup):** Handles the embedding of documents into Pinecone for retrieval purposes (not in the main message flow but essential for setup).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures user input messages sent via Telegram and triggers the workflow.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Sticky Note (Telegram Message Trigger)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger node  
    - *Technical Role:* Listens for incoming Telegram messages to initiate workflow execution.  
    - *Configuration:* Listens for "message" updates only. No filters on chatId or message type.  
    - *Input/Output:* No input; outputs the message JSON including user text and chat metadata.  
    - *Potential Failures:* Telegram API rate limits, authentication errors, or webhook misconfigurations.  
    - *Sticky Note:* "Telegram Message Trigger"

  - **Sticky Note**  
    - *Type:* Visual annotation  
    - *Purpose:* Labels the input reception block as “Telegram Message Trigger.”  
    - *No inputs or outputs.*

#### 2.2 Contextual Memory Management

- **Overview:**  
  Maintains short-term conversational context for each user chat session to enable memory-aware AI responses.

- **Nodes Involved:**  
  - Simple Memory1

- **Node Details:**

  - **Simple Memory1**  
    - *Type:* LangChain MemoryBufferWindow node  
    - *Technical Role:* Stores and manages conversation history by session key (Telegram chat ID), maintaining a context window of the last 10 entries.  
    - *Configuration:*  
      - `sessionKey` is set dynamically from Telegram chat ID.  
      - `contextWindowLength` = 10 messages to keep recent context.  
    - *Input:* Receives AI Agent outputs (conversation turns).  
    - *Output:* Provides memory context back to AI Agent for improved response continuity.  
    - *Potential Failures:* Session key extraction errors, memory overflow or data loss under heavy load.  

#### 2.3 AI Processing

- **Overview:**  
  Processes the user’s question with retrieval-augmented generation using OpenAI's GPT-4o-mini model, referencing relevant legal documents via Pinecone vector search.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Legal Contract Library (Pinecone vector store)  
  - Embeddings OpenAI  
  - Sticky Note (Legal Assistant AI Agent)

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Technical Role:* Central AI component that accepts user input, invokes language model and retrieval tools, and generates final responses.  
    - *Configuration:*  
      - Input text taken from incoming Telegram message `message.text`.  
      - System message defines the AI as a legal assistant that must refer to Pinecone "Legal Contract Library" for factual answers. It explicitly instructs not to fabricate facts and to admit lack of information if necessary.  
      - Uses prompt type “define” to customize system prompt.  
    - *Inputs:* Text from Telegram Trigger, memory from Simple Memory1, language model from OpenAI Chat Model, retrieval tool from Legal Contract Library.  
    - *Outputs:* Final AI-generated answer text, passed to Telegram node.  
    - *Potential Failures:* API limits, incorrect prompt formatting, retrieval tool failures, or memory context mismatches.  
    - *Sticky Note:* "Legal Assistant AI Agent"

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model node  
    - *Technical Role:* Provides GPT-4o-mini large language model capabilities for text generation.  
    - *Configuration:* Model set to "gpt-4o-mini" with no additional options.  
    - *Credentials:* Requires valid OpenAI API credentials.  
    - *Input:* Connected as AI Agent’s language model.  
    - *Output:* Generated text completions for AI Agent.  
    - *Potential Failures:* OpenAI API key issues, model availability, network timeouts.

  - **Legal Contract Library**  
    - *Type:* LangChain Pinecone Vector Store node  
    - *Technical Role:* Retrieves relevant legal documents or clauses from a Pinecone index as a tool for AI Agent.  
    - *Configuration:*  
      - Mode set to "retrieve-as-tool" to integrate as a retrieval tool inside AI Agent.  
      - Uses a specific Pinecone index (name redacted).  
      - Tool description clarifies it extracts contracts, legal policies, and regulatory docs.  
    - *Credentials:* Requires Pinecone API credentials.  
    - *Input:* Receives vector embeddings from Embeddings OpenAI.  
    - *Output:* Returns retrieval results to AI Agent.  
    - *Potential Failures:* Vector index unavailability, Pinecone API issues, embeddings mismatch.

  - **Embeddings OpenAI**  
    - *Type:* LangChain OpenAI Embeddings node  
    - *Technical Role:* Creates vector embeddings for documents or queries to facilitate semantic search in Pinecone.  
    - *Configuration:* Uses default OpenAI embedding options.  
    - *Credentials:* OpenAI API key required.  
    - *Input:* Feeds embeddings into Legal Contract Library.  
    - *Potential Failures:* API rate limits, embedding errors.

  - **Sticky Note (Legal Assistant AI Agent)**  
    - *Content:* Describes the block as the core AI processing with retrieval and RAG setup.

#### 2.4 Output Delivery

- **Overview:**  
  Sends the AI-generated response back to the user on Telegram.

- **Nodes Involved:**  
  - Telegram  
  - Sticky Note (Telegram Output)

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram node (send message)  
    - *Technical Role:* Posts the AI response text into the Telegram chat corresponding to the user’s chat ID.  
    - *Configuration:*  
      - Text to send is taken from AI Agent’s output property `output`.  
      - `chatId` must be set dynamically (empty in JSON but should be configured to the Telegram Trigger chat ID during runtime).  
      - Append attribution is disabled for cleaner output.  
    - *Credentials:* Requires Telegram API credentials.  
    - *Input:* AI Agent output.  
    - *Potential Failures:* Telegram API errors, missing or incorrect chat ID, message length limits.

  - **Sticky Note (Telegram Output)**  
    - *Labels* the output delivery block.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                         | Input Node(s)         | Output Node(s)       | Sticky Note                       |
|---------------------|----------------------------------|---------------------------------------|-----------------------|----------------------|----------------------------------|
| Telegram Trigger     | Telegram Trigger                  | Captures incoming Telegram messages   | -                     | AI Agent             | Telegram Message Trigger          |
| Simple Memory1       | LangChain MemoryBufferWindow     | Maintains conversation context        | AI Agent              | AI Agent             |                                  |
| AI Agent            | LangChain Agent                  | Processes input with RAG AI            | Telegram Trigger, Simple Memory1, OpenAI Chat Model, Legal Contract Library | Telegram             | Legal Assistant AI Agent          |
| OpenAI Chat Model    | LangChain OpenAI Chat Model      | Provides GPT-4o-mini language model   | -                     | AI Agent             |                                  |
| Legal Contract Library | LangChain Pinecone Vector Store | Retrieves relevant legal documents    | Embeddings OpenAI      | AI Agent             |                                  |
| Embeddings OpenAI    | LangChain OpenAI Embeddings      | Creates vector embeddings              | -                     | Legal Contract Library |                                  |
| Telegram             | Telegram                         | Sends AI response back to user        | AI Agent               | -                    | Telegram Output                  |
| Sticky Note          | Sticky Note                      | Label for Telegram message trigger    | -                     | -                    | Telegram Message Trigger          |
| Sticky Note1         | Sticky Note                      | Label for AI Agent block               | -                     | -                    | Legal Assistant AI Agent          |
| Sticky Note2         | Sticky Note                      | Label for Telegram output              | -                     | -                    | Telegram Output                  |
| Sticky Note3         | Sticky Note                      | Workflow description and credits       | -                     | -                    | See details in section 5          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen to "message" updates only.  
   - Set credentials with valid Telegram Bot API key.  
   - Position near the workflow start.

2. **Create Simple Memory Buffer node:**  
   - Type: LangChain MemoryBufferWindow  
   - Set `sessionKey` expression to: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Set `contextWindowLength` to 10.  
   - Connect output of AI Agent (to be created) to this node’s input and its output back to AI Agent’s memory input.

3. **Create OpenAI Chat Model node:**  
   - Type: LangChain OpenAI Chat Model  
   - Select model “gpt-4o-mini”.  
   - Add your OpenAI API credentials.  
   - No special options needed.

4. **Create Embeddings OpenAI node:**  
   - Type: LangChain OpenAI Embeddings  
   - Add your OpenAI API credentials.  
   - Use default embedding options.

5. **Create Pinecone Vector Store node (Legal Contract Library):**  
   - Type: LangChain VectorStore Pinecone  
   - Set mode to "retrieve-as-tool".  
   - Specify Pinecone index name connected to your legal documents database.  
   - Add Pinecone API credentials.  
   - Connect Embeddings OpenAI node output to this node’s embedding input.

6. **Create AI Agent node:**  
   - Type: LangChain Agent  
   - Set input text expression to: `={{ $json.message.text }}` (from Telegram Trigger)  
   - Define system message as:  
     `You are a helpful legal assistant. Your task is to answer legal questions of the users by referring to the Pinecone Vector Database "Legal Contract Library" to get an accurate answer.  
      #Rule  
      Do not make up facts, if you do not have the right information, you may say so.`  
   - Set prompt type to “define”.  
   - Connect:  
     - Telegram Trigger output → AI Agent text input  
     - Simple Memory1 output → AI Agent memory input  
     - OpenAI Chat Model output → AI Agent language model input  
     - Legal Contract Library output → AI Agent tool input  
   - Connect AI Agent output to Telegram node input and Simple Memory1 input.

7. **Create Telegram node (send message):**  
   - Type: Telegram  
   - Set `text` to `={{ $json.output }}` (AI Agent’s output)  
   - Dynamically set `chatId` to the Telegram chat ID from incoming message (e.g., `={{ $('Telegram Trigger').item.json.message.chat.id }}`)  
   - Disable "append attribution" for clean messages.  
   - Add Telegram API credentials.  
   - Connect AI Agent output to Telegram node input.

8. **Add Sticky Notes (optional but recommended for clarity):**  
   - Place notes labeling "Telegram Message Trigger", "Legal Assistant AI Agent", and "Telegram Output" near respective nodes.  
   - Add a large sticky note describing the workflow purpose, use cases, and setup instructions.

9. **Test the workflow:**  
   - Activate the workflow.  
   - Send messages to your Telegram bot and verify responses.  
   - Check logs for errors and adjust configurations or credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| 🧑‍⚖️ Legal Assistant Agent — AI-Powered Legal Q&A with Document Retrieval. Category: LegalTech / AI Agent / RAG / Chatbot. This workflow acts as an AI legal assistant chatbot answering queries by retrieving from a vector-indexed legal document library. Powered by OpenAI + Pinecone + Telegram. Ideal for law firms, compliance teams, and legal tech startups.                                                                                                                                                                         | Workflow description sticky note attached to the workflow.                                         |
| For more builds and detailed step-by-step video tutorials, visit Automate with Marc’s YouTube channel: https://www.youtube.com/@Automatewithmarc                                                                                                                                                                                                                                                                                                                                                                                           | Video tutorials and advanced workflow examples.                                                    |
| Setup requires OpenAI, Pinecone, and Telegram credentials. Upload your legal documents into Pinecone vector index for retrieval. Customize system prompts and expand document sources as needed.                                                                                                                                                                                                                                                                                                                                         | Setup instructions overview.                                                                        |
| The workflow uses a retrieval-augmented generation (RAG) approach ensuring factual, memory-aware answers with fallback if information is unavailable. Fully customizable and extendable.                                                                                                                                                                                                                                                                                                                                                  | Design and feature notes.                                                                           |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.