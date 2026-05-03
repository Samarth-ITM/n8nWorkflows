Document Q&A Chatbot with Google Drive, GPT-4-mini & Telegram (RAG System)

https://n8nworkflows.xyz/workflows/document-q-a-chatbot-with-google-drive--gpt-4-mini---telegram--rag-system--8147


# Document Q&A Chatbot with Google Drive, GPT-4-mini & Telegram (RAG System)

### 1. Workflow Overview

This workflow implements a **Document Q&A Chatbot** that integrates **Google Drive**, **OpenAI GPT-4-mini**, and **Telegram** to build a Retrieval-Augmented Generation (RAG) system. It enables users to upload documents to a designated Google Drive folder, which are then ingested, embedded, and stored in an in-memory vector store. Users can interact with the chatbot through Telegram, asking questions that the AI agent answers by retrieving and analyzing relevant documents from the vector store.

The workflow is logically divided into the following blocks:

- **1.1 Document Ingestion & Embedding:** Triggered by file uploads in Google Drive, downloads the file, processes it into documents, embeds content using OpenAI embeddings, and inserts vectors into the in-memory vector store.
- **1.2 User Interaction & Query Processing:** Listens for Telegram messages, manages conversation memory, retrieves relevant documents from the vector store, queries the AI agent powered by GPT-4-mini, and sends answers back via Telegram.
- **1.3 Supporting Utilities:** Nodes that support embedding consistency, memory handling, and notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion & Embedding

**Overview:**  
This block monitors a specific Google Drive folder for new files, downloads those files, converts them into text documents, generates embeddings via OpenAI, and inserts these into an in-memory vector store for later similarity searches.

**Nodes Involved:**  
- File uploaded (Google Drive Trigger)  
- Download file (Google Drive)  
- Default Data Loader (LangChain Document Loader)  
- Embedding model (OpenAI Embeddings)  
- Insert documents (Vector Store in Memory)  

**Node Details:**

- **File uploaded**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches a specific Google Drive folder (`Folder ID: 1Nc_T_wHj8eF6LLed8D8hZX-q1YUeZT5j`) for new files created.  
  - *Config:* Polls every minute, triggers on newly created files only. Requires Google Drive OAuth2 credentials.  
  - *Connections:* Outputs to `Download file`.  
  - *Edge Cases:* Potential latency up to 1 minute due to polling; missing permissions may cause auth errors.

- **Download file**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the file using the Google Drive file ID from the trigger node.  
  - *Config:* Operation set to download; uses OAuth2 credentials.  
  - *Connections:* Outputs to `Insert documents`.  
  - *Edge Cases:* File may be large or unsupported format; download failures possible due to network or permissions.

- **Default Data Loader**  
  - *Type:* LangChain Document Default Data Loader  
  - *Role:* Converts the binary file data into text documents compatible with LangChain.  
  - *Config:* DataType set to binary for correct handling.  
  - *Connections:* Feeds into `Insert documents` via the AI document input.  
  - *Edge Cases:* Unsupported file formats may cause parsing errors.

- **Embedding model**  
  - *Type:* LangChain Embeddings OpenAI node  
  - *Role:* Generates embeddings for the document text using the OpenAI API.  
  - *Config:* Uses OpenAI API credentials; default embedding settings.  
  - *Connections:* Outputs embeddings to `Insert documents` and `Retrieve documents` nodes to maintain embedding consistency.  
  - *Edge Cases:* Rate limits or API errors; mismatch if embeddings differ between insert and retrieve.

- **Insert documents**  
  - *Type:* LangChain Vector Store In Memory node  
  - *Role:* Inserts document embeddings into an in-memory vector store keyed by `vector_store_key`.  
  - *Config:* Mode set to `insert`; memory key configured to persist vectors.  
  - *Connections:* Receives inputs from `Download file`, `Default Data Loader`, and `Embedding model`.  
  - *Edge Cases:* Memory store resets on workflow restart; large data volume may exhaust memory.

---

#### 2.2 User Interaction & Query Processing

**Overview:**  
This block handles incoming user messages from Telegram, maintains conversational context with memory, retrieves relevant documents via similarity search, processes the query through an AI agent using GPT-4-mini, and returns the response back to the user on Telegram.

**Nodes Involved:**  
- Listen for incoming events (Telegram Trigger)  
- Simple Memory (LangChain Buffer Window Memory)  
- Retrieve documents (LangChain Vector Store Retrieval)  
- AI Agent (LangChain Agent)  
- Model (LangChain Chat OpenAI GPT-4-mini)  
- Telegram (Telegram node for sending messages)  

**Node Details:**

- **Listen for incoming events**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for new Telegram messages (`message` update type).  
  - *Config:* Webhook configured with Telegram credentials; triggers on any new message.  
  - *Connections:* Outputs to `AI Agent`.  
  - *Edge Cases:* Webhook misconfiguration; message types other than text not handled.

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains conversation history per chat session using Telegram chat ID as session key.  
  - *Config:* Session key set dynamically from `message.chat.id` for user-specific memory.  
  - *Connections:* Supplies memory context to `AI Agent`.  
  - *Edge Cases:* Memory buffer size not specified; may cause loss of long-term context.

- **Retrieve documents**  
  - *Type:* LangChain Vector Store In Memory (Retrieve mode)  
  - *Role:* Searches the vector store keyed by `vector_store_key` for top 10 relevant documents based on similarity to user query.  
  - *Config:* Mode set to retrieve-as-tool; includes a tool description to guide retrieval.  
  - *Connections:* Outputs to `AI Agent` as a tool input. Also shares embedding settings with `Embedding model`.  
  - *Edge Cases:* Retrieval quality depends on embedding consistency; empty or irrelevant results possible.

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* The core agent that combines user query, retrieved documents, and conversational memory to generate an answer.  
  - *Config:*  
    - Input text from `message.text`  
    - System prompt defines the agent as a "Knowledge Store Agent" with detailed instructions for searching, analyzing, and responding based on vector data.  
    - Uses the `Model` node as its language model.  
    - Utilizes `Simple Memory` for session context and `Retrieve documents` as a tool for data access.  
  - *Connections:* Outputs response text to `Telegram`.  
  - *Edge Cases:* Complex prompt may cause token limit issues; API errors; incomplete or ambiguous user queries.

- **Model**  
  - *Type:* LangChain Chat OpenAI (GPT-4-mini)  
  - *Role:* Provides the language model backend for the AI Agent.  
  - *Config:* Model set to `gpt-4.1-mini`, using OpenAI API credentials.  
  - *Connections:* Connected as the AI language model input to `AI Agent`.  
  - *Edge Cases:* API rate limits; model version availability.

- **Telegram**  
  - *Type:* Telegram node  
  - *Role:* Sends AI-generated answers back to the user on Telegram.  
  - *Config:*  
    - Text set dynamically from AI Agent output.  
    - Chat ID extracted from incoming Telegram message (`from.id`).  
    - Markdown formatting enabled, no attribution appended.  
    - Error handling set to continue on failure.  
  - *Connections:* Receives input from `AI Agent`.  
  - *Edge Cases:* Delivery failures if user blocked bot or network issues.

---

#### 2.3 Supporting Utilities

**Overview:**  
This block consists of notes and guidance nodes that provide information about the workflow setup, embedding consistency recommendations, and usage instructions.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note3  

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Provides a detailed description of the knowledge store agent, setup instructions for Google Drive and OpenAI credentials, folder configuration, and next steps.  
  - *Content Highlights:*  
    - Explains how to configure credentials and folder.  
    - Instructions to upload files and execute the workflow.  
    - Suggests expanding to other data sources.  
  - *Position:* Visible near the start of the workflow for user reference.

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Emphasizes the importance of using the same embedding model node for both insert and retrieve operations to ensure consistent vector representations.  
  - *Content Highlights:*  
    - Warns about potential issues with mismatched embeddings.  
  - *Position:* Near embedding-related nodes as a reminder.

---

### 3. Summary Table

| Node Name                | Node Type                                   | Functional Role                              | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                                           |
|--------------------------|---------------------------------------------|----------------------------------------------|----------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------|
| File uploaded            | Google Drive Trigger                         | Trigger on new files in Google Drive folder  |                            | Download file            | See Sticky Note: Setup instructions for Google Drive folder and workflow usage                                        |
| Download file            | Google Drive                               | Download files from Google Drive              | File uploaded              | Insert documents         |                                                                                                                       |
| Default Data Loader      | LangChain Document Default Data Loader      | Convert file binary to documents              |                            | Insert documents         |                                                                                                                       |
| Embedding model          | LangChain Embeddings OpenAI                  | Generate embeddings for documents             |                            | Insert documents, Retrieve documents | See Sticky Note3: Embedding consistency required for insert and retrieve operations                                    |
| Insert documents         | LangChain Vector Store In Memory             | Insert document embeddings into vector store | Download file, Default Data Loader, Embedding model |                         |                                                                                                                       |
| Listen for incoming events| Telegram Trigger                             | Listen for Telegram messages                   |                            | AI Agent                 |                                                                                                                       |
| Simple Memory            | LangChain Memory Buffer Window               | Maintain conversation memory                   |                            | AI Agent                 |                                                                                                                       |
| Retrieve documents       | LangChain Vector Store In Memory (Retrieve) | Retrieve relevant documents for queries       | Embedding model            | AI Agent                 |                                                                                                                       |
| AI Agent                 | LangChain Agent                             | Process user query with retrieved docs & memory | Listen for incoming events, Simple Memory, Retrieve documents, Model | Telegram                 |                                                                                                                       |
| Model                    | LangChain Chat OpenAI (GPT-4-mini)           | Language model backend for AI Agent            |                            | AI Agent                 |                                                                                                                       |
| Telegram                 | Telegram node                              | Send answers back to Telegram                   | AI Agent                   |                         |                                                                                                                       |
| Sticky Note              | Sticky Note                                | Workflow overview and setup instructions       |                            |                         | See Sticky Note: Setup instructions for Google Drive and OpenAI credentials, folder configuration                    |
| Sticky Note3             | Sticky Note                                | Embedding consistency reminder                  |                            |                         | See Sticky Note3: Embedding node must be identical for insert and retrieve                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node (File uploaded):**  
   - Type: Google Drive Trigger  
   - Configure:  
     - Event: `fileCreated`  
     - Poll interval: every minute  
     - Trigger on: specific folder (enter folder ID: `1Nc_T_wHj8eF6LLed8D8hZX-q1YUeZT5j`)  
   - Credentials: Set Google Drive OAuth2 credentials  
   - Connect output to `Download file`.

2. **Create Google Drive Node (Download file):**  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: set dynamically from trigger node's file ID (`={{ $json.id }}`)  
   - Credentials: Same Google Drive OAuth2 credentials  
   - Connect output to `Insert documents`.

3. **Create LangChain Document Loader Node (Default Data Loader):**  
   - Type: LangChain Document Default Data Loader  
   - Data Type: `binary` (to process downloaded file content)  
   - Connect output to `Insert documents` as AI Document input.

4. **Create LangChain Embeddings Node (Embedding model):**  
   - Type: LangChain Embeddings OpenAI  
   - Use default embedding settings  
   - Credentials: OpenAI API credentials  
   - Connect outputs to both `Insert documents` and `Retrieve documents` nodes to ensure embedding consistency.

5. **Create Vector Store Node for Insert (Insert documents):**  
   - Type: LangChain Vector Store In Memory  
   - Mode: `insert`  
   - Memory Key: `vector_store_key` (to persist vectors in-memory)  
   - Connect inputs from `Download file`, `Default Data Loader`, and `Embedding model`.

6. **Create Telegram Trigger Node (Listen for incoming events):**  
   - Type: Telegram Trigger  
   - Updates: `message`  
   - Credentials: Telegram API credentials  
   - Connect output to `AI Agent`.

7. **Create LangChain Memory Node (Simple Memory):**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: set to Telegram chat ID (`={{ $json.message.chat.id }}`)  
   - Connect output to `AI Agent` as AI Memory input.

8. **Create Vector Store Node for Retrieve (Retrieve documents):**  
   - Type: LangChain Vector Store In Memory  
   - Mode: `retrieve-as-tool`  
   - Top K: 10 (return top 10 results)  
   - Memory Key: `vector_store_key` (same as insert node)  
   - Tool Description: `"Use this tool to retrieve any information required."`  
   - Connect output as AI Tool input to `AI Agent`.

9. **Create LangChain Agent Node (AI Agent):**  
   - Type: LangChain Agent  
   - Text input: `={{ $json.message.text }}` (user's Telegram message)  
   - System Message: Use the detailed Knowledge Store Agent prompt provided in the original workflow (see Section 2.2 for full prompt)  
   - Prompt type: `define`  
   - Connect AI Language Model input from `Model` node  
   - Connect AI Memory input from `Simple Memory`  
   - Connect AI Tool input from `Retrieve documents`  
   - Connect output to `Telegram` node.

10. **Create LangChain Chat OpenAI Node (Model):**  
    - Type: LangChain Chat OpenAI  
    - Model: `gpt-4.1-mini`  
    - Credentials: OpenAI API credentials  
    - Connect output as AI Language Model input to `AI Agent`.

11. **Create Telegram Node (Telegram):**  
    - Type: Telegram  
    - Text: `={{ $json.output }}` (response from AI Agent)  
    - Chat ID: `={{ $('Listen for incoming events').first().json.message.from.id }}` (send to user)  
    - Additional Fields:  
      - Parse mode: Markdown  
      - Append attribution: false  
    - Error handling: Continue on error  
    - Connect input from `AI Agent`.

12. **Add Sticky Notes (optional):**  
    - Add notes for setup instructions and embedding consistency reminders near their relevant nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Knowledge store agent is a chat-based AI agent using documents uploaded to Google Drive and stored in a vector database. Setup requires Google Drive and OpenAI credentials, and a dedicated folder for document uploads.       | Workflow Sticky Note                                                                                |
| Embedding node must be the same for insert and retrieve operations to avoid retrieval failures due to embedding mismatch.                                                                                                      | Workflow Sticky Note3                                                                               |
| GPT-4-mini model is used for query answering to balance response quality and resource use. Ensure OpenAI API keys have access to this model.                                                                                   | Model Node Configuration                                                                            |
| Polling for new files in Google Drive has a latency of up to one minute; consider alternative triggers or webhook setups for more real-time ingestion if needed.                                                                | Google Drive Trigger Node Configuration                                                           |
| The AI Agent system prompt is detailed and guides the agent to search, analyze, and respond based strictly on retrieved vector data, emphasizing transparency and honesty regarding data availability.                            | AI Agent Node Configuration                                                                        |
| Telegram webhook must be correctly configured for inbound messages to trigger the workflow; ensure bot permissions and webhook URLs are properly set.                                                                           | Telegram Trigger Node Configuration                                                               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.