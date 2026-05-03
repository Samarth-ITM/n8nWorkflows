Chat with Documents via RAG: Google Drive to GPT-5 with Supabase Vector Database

https://n8nworkflows.xyz/workflows/chat-with-documents-via-rag--google-drive-to-gpt-5-with-supabase-vector-database-7308


# Chat with Documents via RAG: Google Drive to GPT-5 with Supabase Vector Database

### 1. Workflow Overview

This n8n workflow implements a Retrieval-Augmented Generation (RAG) chatbot solution enabling users to interact with documents stored on Google Drive via an advanced GPT-5 language model, with vector similarity search powered by Supabase. It is designed for use cases where users need AI-driven conversational access to documents, including scanned files requiring OCR, and supports multi-channel chat triggers (Slack, Telegram, Gmail, WhatsApp).

The workflow is organized into the following logical blocks:

- **1.1 Document Ingestion & Processing:** Detects new Google Drive files, downloads, optionally OCRs, splits text, generates embeddings, and stores them in Supabase vector database.
- **1.2 Vector Store & Embeddings Setup:** Manages embeddings generation and vector store interactions for document indexing and retrieval.
- **1.3 Chat Trigger & Preprocessing:** Listens for incoming chat messages from multiple platforms, standardizes input, and prepares context.
- **1.4 RAG Agent & Language Model Processing:** Executes the RAG agent that queries Supabase with embeddings, integrates chat memory, and leverages GPT-5 for response generation.
- **1.5 Response Delivery:** Sends generated answers back to users via the appropriate messaging platform.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion & Processing

**Overview:**  
This block listens for new files created in Google Drive, downloads them, performs OCR if needed, splits the text into manageable chunks, generates embeddings, and stores data in the Supabase vector database.

**Nodes Involved:**  
- Google Drive Trigger: `Create file`  
- Google Drive: `Download File4`  
- OCR: `OCR`  
- Split Output: `Split Out1`  
- Text Splitter: `Character Text Splitter4`  
- Default Data Loader: `Default Data Loader4`  
- Embeddings OpenAI: `Embeddings OpenAI8`  
- Supabase Vector Store Insert: `Insert into Supabase Vectorstore3`

**Node Details:**

- **Create file**  
  - *Type:* Google Drive Trigger  
  - *Role:* Detects newly created files in Google Drive to initiate ingestion  
  - *Config:* Default trigger, no filters specified  
  - *Connections:* Outputs to `Download File4`  
  - *Potential failures:* API quota limits, network errors
  
- **Download File4**  
  - *Type:* Google Drive (Download)  
  - *Role:* Downloads the file detected by trigger  
  - *Config:* Uses file ID from trigger output  
  - *Connections:* Outputs to `OCR`  
  - *Edge cases:* Unsupported file types, download failures
  
- **OCR**  
  - *Type:* Mistral AI (OCR)  
  - *Role:* Converts image or scanned PDFs to text  
  - *Config:* Default OCR parameters  
  - *Connections:* Outputs to `Split Out1`  
  - *Edge cases:* Poor image quality causing inaccurate OCR
  
- **Split Out1**  
  - *Type:* Split Out  
  - *Role:* Flattens multi-part OCR output for downstream processing  
  - *Connections:* Outputs to `Insert into Supabase Vectorstore3`
  
- **Character Text Splitter4**  
  - *Type:* Langchain Character Text Splitter  
  - *Role:* Splits large texts into smaller chunks for embedding generation  
  - *Config:* Uses character-based splitting strategy (default parameters)  
  - *Connections:* Sends chunks to `Default Data Loader4`
  
- **Default Data Loader4**  
  - *Type:* Langchain Default Data Loader  
  - *Role:* Prepares chunked text for embedding  
  - *Connections:* Outputs to `Insert into Supabase Vectorstore3`
  
- **Embeddings OpenAI8**  
  - *Type:* Langchain OpenAI Embeddings  
  - *Role:* Generates vector embeddings for document chunks  
  - *Config:* Uses OpenAI API with configured credentials  
  - *Connections:* Outputs to `Insert into Supabase Vectorstore3`  
  - *Edge cases:* API rate limits, invalid credentials
  
- **Insert into Supabase Vectorstore3**  
  - *Type:* Langchain Supabase Vector Store  
  - *Role:* Inserts document embeddings and metadata into Supabase vector database  
  - *Config:* Supabase credentials configured  
  - *Input:* Receives embeddings and document metadata  
  - *Output:* None (terminal for ingestion branch)  
  - *Failures:* Network errors, authentication failures, data schema mismatches

---

#### 2.2 Vector Store & Embeddings Setup

**Overview:**  
This block sets up the vector store and embeddings for query-time retrieval and interacts with the RAG agent.

**Nodes Involved:**  
- Supabase Vector Store: `Supabase Vector Store`  
- Embeddings OpenAI: `Embeddings OpenAI6`

**Node Details:**

- **Embeddings OpenAI6**  
  - *Type:* Langchain OpenAI Embeddings  
  - *Role:* Generates embeddings for user queries or documents during retrieval  
  - *Config:* OpenAI API key usage  
  - *Connections:* Outputs to `Supabase Vector Store`
  
- **Supabase Vector Store**  
  - *Type:* Langchain Supabase Vector Store  
  - *Role:* Provides vector similarity search capabilities from Supabase  
  - *Config:* Supabase project URL and API key  
  - *Connections:* Feeds into the `RAG Agent` node for retrieval  
  - *Failures:* Query timeouts, incorrect vector dimensions

---

#### 2.3 Chat Trigger & Preprocessing

**Overview:**  
This block receives chat messages from various messaging platforms, normalizes input, and passes it to the RAG agent.

**Nodes Involved:**  
- Langchain Chat Trigger: `When chat message received`  
- Slack Trigger: `Slack Trigger`  
- Telegram Trigger: `Telegram Trigger`  
- Gmail Trigger: `Gmail Trigger`  
- WhatsApp Trigger: `WhatsApp Trigger` (disabled)  
- Edit Fields: `Edit Fields`

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Listens for incoming chat messages (main webhook entrypoint)  
  - *Config:* Webhook ID specified  
  - *Connections:* Outputs to `RAG Agent`
  - *Failures:* Webhook connectivity, invalid message formats
  
- **Slack Trigger, Telegram Trigger, Gmail Trigger, WhatsApp Trigger**  
  - *Type:* Messaging platform specific triggers  
  - *Role:* Provides triggers from respective platforms to capture chat inputs  
  - *Config:* Webhook IDs set for each platform; WhatsApp trigger disabled  
  - *Connections:* All output to `Edit Fields` node for normalization  
  - *Failures:* Auth token expiry, webhook misconfiguration
  
- **Edit Fields**  
  - *Type:* Set (Data Editing)  
  - *Role:* Normalizes incoming message content and metadata before feeding to RAG agent  
  - *Connections:* Outputs to `RAG Agent`

---

#### 2.4 RAG Agent & Language Model Processing

**Overview:**  
Core AI logic combining chat memory, vector store retrieval, and GPT-5 language model to generate contextually relevant answers.

**Nodes Involved:**  
- RAG Agent: `RAG Agent`  
- Chat Memory: `Chat Memory`  
- GPT 5: `GPT 5`

**Node Details:**

- **RAG Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Orchestrates retrieval of relevant document chunks from Supabase and interacts with GPT-5 model  
  - *Inputs:* Receives chat input (from triggers), memory (from `Chat Memory`), and vector store tool (from `Supabase Vector Store`)  
  - *Outputs:* Generated answer routed to response delivery nodes  
  - *Failures:* Query failures, model API limits, memory access errors
  
- **Chat Memory**  
  - *Type:* Langchain Postgres Chat Memory  
  - *Role:* Stores and retrieves conversational context to maintain dialogue coherence  
  - *Config:* Connects to a Postgres database for persistent memory  
  - *Connections:* Feeds memory context to `RAG Agent`
  
- **GPT 5**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Language model generating natural language responses  
  - *Config:* Uses GPT-5 via OpenAI API with proper credentials  
  - *Connections:* Connected as language model for `RAG Agent`  
  - *Failures:* API rate limits, malformed input, timeout

---

#### 2.5 Response Delivery

**Overview:**  
Delivers the AI-generated responses back to users via their respective messaging channels.

**Nodes Involved:**  
- Telegram: `Send a text message`  
- WhatsApp: `Send message` (disabled)  
- Slack: `Send a message`  
- Gmail: `Send a message1`

**Node Details:**

- **Send a text message (Telegram)**  
  - *Type:* Telegram Node  
  - *Role:* Sends text reply to Telegram users  
  - *Config:* Requires Telegram bot credentials  
  - *Input:* Receives RAG Agent output  
  - *Failures:* Bot authentication failures, message formatting errors
  
- **Send message (WhatsApp)**  
  - *Type:* WhatsApp Node  
  - *Role:* Intended for WhatsApp message delivery (currently disabled)  
  - *Failures:* WhatsApp API restrictions, disabled node
  
- **Send a message (Slack)**  
  - *Type:* Slack Node  
  - *Role:* Sends response message to Slack channel or user  
  - *Config:* Slack OAuth credentials configured  
  - *Input:* Receives RAG Agent output
  
- **Send a message1 (Gmail)**  
  - *Type:* Gmail Node  
  - *Role:* Sends email responses via Gmail  
  - *Config:* OAuth2 credentials for Gmail configured  
  - *Input:* Receives RAG Agent output

---

### 3. Summary Table

| Node Name                   | Node Type                                 | Functional Role                          | Input Node(s)                  | Output Node(s)                                 | Sticky Note                              |
|-----------------------------|-------------------------------------------|----------------------------------------|-------------------------------|-----------------------------------------------|------------------------------------------|
| Create file                 | Google Drive Trigger                       | Detect new Google Drive files           | None                          | Download File4                                |                                          |
| Download File4              | Google Drive                              | Download detected file                  | Create file                   | OCR                                           |                                          |
| OCR                        | Mistral AI                               | Perform OCR on downloaded files         | Download File4                | Split Out1                                    |                                          |
| Split Out1                 | Split Out                                | Flatten OCR output                      | OCR                          | Insert into Supabase Vectorstore3              |                                          |
| Character Text Splitter4   | Langchain Text Splitter                   | Split text into chunks                  | None (implied with Default Data Loader4) | Default Data Loader4                      |                                          |
| Default Data Loader4       | Langchain Default Data Loader             | Prepare documents for embedding        | Character Text Splitter4      | Insert into Supabase Vectorstore3              |                                          |
| Embeddings OpenAI8         | Langchain OpenAI Embeddings                | Generate embeddings for document chunks | Default Data Loader4          | Insert into Supabase Vectorstore3              |                                          |
| Insert into Supabase Vectorstore3 | Langchain Supabase Vector Store            | Store embeddings in Supabase vector DB  | Split Out1, Embeddings OpenAI8, Default Data Loader4 | None                          |                                          |
| Supabase Vector Store      | Langchain Supabase Vector Store            | Vector store for RAG retrieval          | Embeddings OpenAI6           | RAG Agent                                     |                                          |
| Embeddings OpenAI6         | Langchain OpenAI Embeddings                | Generate query embeddings                | None                        | Supabase Vector Store                          |                                          |
| When chat message received | Langchain Chat Trigger                    | Entry webhook for chat messages         | None                        | RAG Agent                                     |                                          |
| Slack Trigger              | Slack Trigger                            | Trigger on Slack messages                | None                        | Edit Fields                                   |                                          |
| Telegram Trigger           | Telegram Trigger                         | Trigger on Telegram messages             | None                        | Edit Fields                                   |                                          |
| Gmail Trigger              | Gmail Trigger                            | Trigger on Gmail messages                | None                        | Edit Fields                                   |                                          |
| WhatsApp Trigger           | WhatsApp Trigger (disabled)              | Trigger on WhatsApp messages             | None                        | Edit Fields                                   | Disabled node                            |
| Edit Fields                | Set                                      | Normalize incoming chat fields           | Slack Trigger, Telegram Trigger, Gmail Trigger, WhatsApp Trigger | RAG Agent                     |                                          |
| Chat Memory                | Langchain Postgres Chat Memory             | Store and retrieve chat conversation context | None                        | RAG Agent                                     |                                          |
| RAG Agent                 | Langchain Agent                          | Retrieve relevant docs and generate AI response | Edit Fields, Chat Memory, Supabase Vector Store, GPT 5 | Send a text message, Send message, Send a message, Send a message1 |                                          |
| GPT 5                      | Langchain OpenAI Chat Model                | Generate natural language responses      | RAG Agent (ai_languageModel) | RAG Agent                                     |                                          |
| Send a text message        | Telegram                                | Send response to Telegram users          | RAG Agent                   | None                                          |                                          |
| Send message               | WhatsApp (disabled)                      | Send response to WhatsApp users           | RAG Agent                   | None                                          | Disabled node                            |
| Send a message             | Slack                                   | Send response to Slack channels/users    | RAG Agent                   | None                                          |                                          |
| Send a message1            | Gmail                                   | Send response emails                      | RAG Agent                   | None                                          |                                          |
| Sticky Note                | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          | Multiple sticky notes with empty content |
| Sticky Note1               | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          |                                          |
| Sticky Note2               | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          |                                          |
| Sticky Note3               | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          |                                          |
| Sticky Note4               | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          |                                          |
| Sticky Note5               | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          |                                          |
| Sticky Note6               | Sticky Note                             | Visual notes (empty content)              | None                        | None                                          |                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node (`Create file`):**  
   - Configure it to trigger on new file creation events in your Google Drive account.

2. **Add a Google Drive node (`Download File4`):**  
   - Connect input from `Create file`.  
   - Configure to download the file using the file ID from the trigger.

3. **Add an OCR node (`OCR`):**  
   - Connect input from `Download File4`.  
   - Configure with Mistral AI or your preferred OCR service to extract text from downloaded files.

4. **Add a Split Out node (`Split Out1`):**  
   - Connect input from `OCR`.  
   - This node flattens OCR output arrays for further processing.

5. **Add a Character Text Splitter node (`Character Text Splitter4`):**  
   - Connect logically to the document processing flow (after OCR or download).  
   - Configure to split long text into smaller chunks (default settings suffice).

6. **Add a Default Data Loader node (`Default Data Loader4`):**  
   - Connect input from `Character Text Splitter4`.  
   - Prepares document chunks for embeddings.

7. **Add an Embeddings OpenAI node (`Embeddings OpenAI8`):**  
   - Connect input from `Default Data Loader4`.  
   - Set API key credentials for OpenAI. Use embedding model (e.g., text-embedding-ada-002).

8. **Add a Supabase Vector Store node for insertion (`Insert into Supabase Vectorstore3`):**  
   - Connect inputs from `Split Out1`, `Embeddings OpenAI8`, and `Default Data Loader4`.  
   - Configure with your Supabase project URL and API key for vector store insertion.

9. **Add an Embeddings OpenAI node for queries (`Embeddings OpenAI6`):**  
   - Configure similarly to the ingestion embeddings node.  
   - Connect output to the Supabase Vector Store retrieval node.

10. **Add a Supabase Vector Store node for retrieval (`Supabase Vector Store`):**  
    - Connect input from `Embeddings OpenAI6`.  
    - Configure with Supabase credentials, set as vector search tool.

11. **Add a Langchain Chat Trigger node (`When chat message received`):**  
    - Configure webhook for incoming chat messages.

12. **Add platform-specific trigger nodes:**  
    - Slack Trigger, Telegram Trigger, Gmail Trigger (enable as needed).  
    - Configure respective webhook IDs and credentials.

13. **Add an Edit Fields node (`Edit Fields`):**  
    - Connect inputs from all platform triggers.  
    - Normalize and standardize incoming message data before feeding to RAG agent.

14. **Add a Langchain Postgres Chat Memory node (`Chat Memory`):**  
    - Configure connection to a Postgres database for chat context persistence.

15. **Add a GPT-5 Langchain Chat Model node (`GPT 5`):**  
    - Configure with OpenAI GPT-5 credentials and parameters.

16. **Add a Langchain Agent node (`RAG Agent`):**  
    - Connect inputs from `Edit Fields` (main input), `Chat Memory` (memory), `Supabase Vector Store` (tool), and `GPT 5` (language model).  
    - Configure agent to perform retrieval augmented generation.

17. **Add response delivery nodes:**  
    - Telegram node (`Send a text message`), Slack node (`Send a message`), Gmail node (`Send a message1`).  
    - Connect all to `RAG Agent` output.  
    - Configure credentials for each messaging platform.

18. **(Optional) Disable WhatsApp trigger and message sending nodes if not used.**

19. **Test the entire flow:**  
    - Upload documents to Google Drive to validate ingestion.  
    - Send chat messages via configured platforms to test RAG responses.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow leverages Langchain n8n nodes extensively for AI document processing and chat.   | See https://js.langchain.com/docs/use/n8n/                          |
| Supabase is used as a vector database for fast semantic search.                               | https://supabase.com/docs/guides/database/vector-search             |
| GPT-5 is accessed via OpenAI API—ensure API keys have correct access and quota.               | https://platform.openai.com/docs/models/gpt-5                        |
| The OCR node uses Mistral AI; alternative OCR services can be configured as needed.           | https://mistral.ai/                                                 |
| Multi-channel chat triggers enable broad user access; disable unused triggers to optimize performance.| -                                                                  |
| For persistent chat memory, a Postgres database is required and should be configured securely.| https://www.postgresql.org/docs/current/index.html                   |
| n8n documentation for node-specific configurations is available at https://docs.n8n.io/       | -                                                                  |

---

**Disclaimer:** The text provided is extracted solely from an automated n8n workflow. It complies fully with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.