WhatsApp Support Bot with Google Drive RAG, GPT-4.1-mini and Cohere Reranking

https://n8nworkflows.xyz/workflows/whatsapp-support-bot-with-google-drive-rag--gpt-4-1-mini-and-cohere-reranking-9043


# WhatsApp Support Bot with Google Drive RAG, GPT-4.1-mini and Cohere Reranking

---
### 1. Workflow Overview

This workflow implements a **WhatsApp Support Bot** for HolistiCare (a multi-specialty clinic in Lahore) that uses **Google Drive for knowledge base ingestion**, **Supabase vector search for retrieval-augmented generation (RAG)**, **OpenAI GPT-4.1-mini for AI chat responses**, and **Cohere for reranking** of search results. Its main purpose is to provide empathetic, concise, and professional customer support replies on WhatsApp, leveraging a continuously updated knowledge base from Google Drive documents.

The workflow is logically divided into two main blocks:

**1.1 Google Drive Document Ingestion Block**  
- Watches a specific Google Drive folder for file updates, downloads and converts updated files to plain text, splits documents into chunks, generates embeddings via OpenAI, and inserts them into a Supabase vector store.  
- This keeps the knowledge base fresh and searchable for the AI agent.

**1.2 WhatsApp AI Agent Interaction Block**  
- Receives incoming WhatsApp messages (text or audio).  
- For audio, downloads and transcribes the message using OpenAI Whisper.  
- Normalizes message fields and merges both text and transcribed audio paths.  
- Uses an AI Agent powered by OpenAI Chat Model (GPT-4.1-mini), with memory for context and a Supabase vector store tool for knowledge retrieval augmented by Cohere reranking.  
- Enforces guardrails on tone, length, and content to provide concise, empathetic, and appropriate replies.  
- Sends the AI-generated reply back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Google Drive Document Ingestion Block

- **Overview:**  
  This block automatically ingests updated Google Drive documents into a Supabase vector store for semantic search. It converts Google Docs to plain text, splits large text files into chunks, generates embeddings, and inserts them into the vector store, enabling up-to-date knowledge retrieval for the AI agent.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Edit Fields  
  - Download file  
  - Character Text Splitter  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Supabase Vector Store  
  - Sticky Note (documentation)

- **Node Details:**  

  - **Google Drive Trigger**  
    - Type: Trigger node for Google Drive events  
    - Configuration: Watches a specific folder (ID `1JCjixUoufxZRiFatfq-Nihr_vYRVoo2r`) for `fileUpdated` events, polling weekly on Sundays.  
    - Input: n/a (event-based)  
    - Output: Emits file metadata of updated files  
    - Failure cases: OAuth token expiry, folder permission errors, no updates detected  
    - Credentials: Google Drive OAuth2  
      
  - **Edit Fields**  
    - Type: Data transformation node (Set)  
    - Configuration: Extracts and sets `id` field from the incoming file metadata for downstream use.  
    - Input: from Google Drive Trigger  
    - Output: JSON including `id` for file download  
    - Failures: Expression failures if input lacks `id`  
      
  - **Download file**  
    - Type: Google Drive node (download operation)  
    - Configuration: Downloads file by `id`, converts Google Docs to plain text (`text/plain`), other formats downloaded as is.  
    - Input: file ID from Edit Fields  
    - Output: Binary data of file content  
    - Failures: Download errors, unsupported file types, conversion failures  
    - Credentials: Google Drive OAuth2  
      
  - **Character Text Splitter**  
    - Type: LangChain text splitter  
    - Configuration: Splits text on period `"."` with chunk size 2000 characters and overlap 300.  
    - Input: text from Default Data Loader  
    - Output: text chunks for embedding  
    - Failures: Empty text, incorrect splitting parameters  
      
  - **Default Data Loader**  
    - Type: LangChain document loader  
    - Configuration: Loads binary data, uses custom text splitting mode (Character Text Splitter)  
    - Input: binary file content from Download file  
    - Output: text documents/chunks for embedding  
    - Failures: Corrupt binary data, loader misconfiguration  
      
  - **Embeddings OpenAI**  
    - Type: LangChain embedding node using OpenAI  
    - Configuration: Uses OpenAI API with default embedding model; credential is set  
    - Input: text chunks from Default Data Loader  
    - Output: embeddings for each chunk  
    - Failures: API key invalid, rate limits, network errors  
    - Credentials: OpenAI API  
      
  - **Supabase Vector Store**  
    - Type: LangChain vector store node (Supabase)  
    - Configuration: Mode set to `insert`, target table `documents`, uses an RPC function `match_documents` for queries  
    - Input: embeddings from Embeddings OpenAI  
    - Output: confirmation of inserted records  
    - Failures: Supabase auth errors, schema mismatches, network errors  
    - Credentials: Supabase API key and URL  
      
  - **Sticky Note**  
    - Purpose: Documentation for adding documents to vector store  
    - Content: Explains process and configuration tips  

---

#### 2.2 WhatsApp AI Agent Interaction Block

- **Overview:**  
  This block handles inbound WhatsApp messages, processes text or audio inputs, normalizes data, queries a Supabase-powered knowledge base with Cohere reranking, uses the OpenAI GPT-4.1-mini model with memory and guardrails to generate short, empathetic replies, and sends replies back to WhatsApp users.

- **Nodes Involved:**  
  - WhatsApp Trigger  
  - Switch  
  - HTTP Request (media metadata fetch)  
  - HTTP Request1 (media download)  
  - Translate a recording (OpenAI transcription)  
  - Merge  
  - Edit Fields1  
  - OpenAI Chat Model  
  - Simple Memory  
  - Kknowledge_base (Supabase vector store query)  
  - Reranker Cohere  
  - AI Agent  
  - Send message  
  - Sticky Notes (documentation)

- **Node Details:**  

  - **WhatsApp Trigger**  
    - Type: Webhook trigger node for WhatsApp Business API  
    - Configuration: Subscribed to `messages` updates via Meta webhook  
    - Input: Incoming WhatsApp messages (text or audio)  
    - Output: JSON with message data including type, sender, message content  
    - Failures: Webhook verification issues, network downtime, auth errors  
    - Credentials: WhatsApp API credentials (App ID, Secret, Verify Token)  
      
  - **Switch**  
    - Type: Conditional routing node  
    - Configuration: Checks `messages[0].type` to branch flow: `text` or `audio`  
    - Input: WhatsApp Trigger output  
    - Output: Routes text messages to Merge, audio messages to HTTP Request  
    - Failures: Unrecognized message types not handled  
      
  - **HTTP Request**  
    - Type: HTTP GET request  
    - Configuration: Fetches media metadata from Facebook Graph API using audio message ID and access token  
    - Input: audio message ID from WhatsApp Trigger  
    - Output: JSON with media URL  
    - Failures: Invalid or expired access tokens, API errors  
      
  - **HTTP Request1**  
    - Type: HTTP GET request  
    - Configuration: Downloads audio file using URL from previous node, sends `Authorization: Bearer <PAGE_ACCESS_TOKEN>` header  
    - Input: media URL from HTTP Request  
    - Output: binary audio file  
    - Failures: Authorization errors, network issues  
      
  - **Translate a recording**  
    - Type: OpenAI audio transcription node (Whisper)  
    - Configuration: Translates audio binary to text  
    - Input: binary audio from HTTP Request1  
    - Output: transcription text  
    - Failures: Transcription errors, API limits  
    - Credentials: OpenAI API  
      
  - **Merge**  
    - Type: Merge node  
    - Configuration: Merges two flows: text messages and audio transcriptions into one unified flow  
    - Input: text from Switch or transcription from Translate a recording  
    - Output: unified JSON with text content field  
    - Failures: Merge conflicts if inputs missing  
      
  - **Edit Fields1**  
    - Type: Data transformation (Set)  
    - Configuration: Sets two fields:  
      - `Phone` = sender phone number (`messages[0].from`)  
      - `text ` = concatenation of transcription text and typed text (`$json.text` + `$json.messages[0].text.body`) for unified message text  
    - Input: merged message data  
    - Output: normalized message JSON for AI processing  
    - Failures: Missing fields in input JSON  
      
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI chat language model node  
    - Configuration: Model `gpt-4.1-mini`, temperature 0.3 for controlled creativity  
    - Input: message text from Edit Fields1  
    - Output: language model response data (for AI Agent)  
    - Failures: API rate limits, timeout, invalid credentials  
    - Credentials: OpenAI API  
      
  - **Simple Memory**  
    - Type: LangChain memory buffer window  
    - Configuration: Session key set dynamically to the user's phone number (`{{$json.Phone}}`) for per-user short-term context memory  
    - Input: memory context updates from AI Agent  
    - Output: memory context injected into AI Agent  
    - Failures: Memory overflow if too long, session key errors  
      
  - **Kknowledge_base**  
    - Type: LangChain vector store query with Supabase  
    - Configuration: Mode `retrieve-as-tool`, topK=10 documents, uses RPC `match_documents`, integrated with Cohere reranker, tool description tailored to clinic queries  
    - Input: query text from AI Agent  
    - Output: relevant documents as grounding knowledge for AI Agent  
    - Failures: Supabase connection errors, empty results, vector embedding mismatches  
    - Credentials: Supabase API  
      
  - **Reranker Cohere**  
    - Type: LangChain reranker node using Cohere API  
    - Configuration: Reranks Supabase retrieval results to improve relevance  
    - Input: retrieval results from Supabase KB  
    - Output: reranked results passed back to KB and AI Agent  
    - Failures: API key errors, rate limiting  
    - Credentials: Cohere API  
      
  - **AI Agent**  
    - Type: LangChain agent node combining language model, memory, and retrieval tools  
    - Configuration:  
      - Receives unified message text  
      - System prompt defines tone, context, guardrails (≤100 words, friendly, no prices unless in KB, no unsolicited appointments, empathetic) specific to HolistiCare  
      - Integrates Simple Memory and Kknowledge_base as tool  
      - On error: continues with regular output  
    - Input: message text, memory context, retrieved knowledge documents  
    - Output: final reply text to send via WhatsApp  
    - Failures: Prompt failures, tool integration errors, large context overflow  
      
  - **Send message**  
    - Type: WhatsApp send message node  
    - Configuration: Sends text message reply (`{{$json.output}}`) to sender phone number from WhatsApp Trigger  
    - Input: AI Agent output text  
    - Output: confirmation of message sent  
    - Failures: WhatsApp API auth errors, invalid phone number, message quota exceeded  
    - Credentials: WhatsApp Business API (phone number ID and access token)  
      
  - **Sticky Notes**  
    - Purpose: Provide detailed documentation and usage instructions for the AI Agent block and WhatsApp integration  

---

### 3. Summary Table

| Node Name             | Node Type                               | Functional Role                            | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                   |
|-----------------------|---------------------------------------|-------------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Google Drive Trigger   | googleDriveTrigger                    | Watches Google Drive folder for updates   | n/a                      | Edit Fields              | See documentation on auto-ingest Google Drive docs into Supabase Vector Store                |
| Edit Fields           | set                                  | Extracts file ID for download              | Google Drive Trigger     | Download file            |                                                                                              |
| Download file         | googleDrive                          | Downloads and converts updated file       | Edit Fields              | Supabase Vector Store    |                                                                                              |
| Character Text Splitter| textSplitterCharacterTextSplitter    | Splits document text into chunks           | Default Data Loader      | Default Data Loader      |                                                                                              |
| Default Data Loader    | documentDefaultDataLoader             | Loads text chunks for embedding            | Character Text Splitter  | Supabase Vector Store    |                                                                                              |
| Embeddings OpenAI     | embeddingsOpenAi                     | Generates embeddings for document chunks   | Default Data Loader      | Supabase Vector Store    |                                                                                              |
| Supabase Vector Store  | vectorStoreSupabase                  | Inserts embeddings into Supabase vector DB | Embeddings OpenAI        |                          |                                                                                              |
| Sticky Note           | stickyNote                          | Documentation for Google Drive ingestion  |                          |                          | ## Add documents to vector store                                                            |
| WhatsApp Trigger       | whatsAppTrigger                     | Receives incoming WhatsApp messages        | n/a                      | Switch                   | See documentation on AI Agent and WhatsApp integration                                      |
| Switch                | switch                             | Routes messages by type (text/audio)       | WhatsApp Trigger         | Merge / HTTP Request     |                                                                                              |
| HTTP Request          | httpRequest                        | Fetches media metadata for audio messages  | Switch (audio branch)    | HTTP Request1            |                                                                                              |
| HTTP Request1         | httpRequest                        | Downloads audio file from media URL         | HTTP Request             | Translate a recording    |                                                                                              |
| Translate a recording  | openAi (audio translate)            | Transcribes audio to text                    | HTTP Request1            | Merge                    |                                                                                              |
| Merge                 | merge                             | Merges text and transcribed audio paths    | Switch, Translate a recording | Edit Fields1          |                                                                                              |
| Edit Fields1           | set                               | Normalizes message fields (Phone, text)    | Merge                    | AI Agent                 |                                                                                              |
| OpenAI Chat Model      | lmChatOpenAi                      | Provides language model for AI Agent       | Edit Fields1             | AI Agent                 |                                                                                              |
| Simple Memory          | memoryBufferWindow                | Maintains short-term memory per user       | AI Agent (memory input)  | AI Agent                 |                                                                                              |
| Kknowledge_base        | vectorStoreSupabase               | Queries Supabase vector store as tool      | AI Agent (tool input)    | AI Agent                 |                                                                                              |
| Reranker Cohere        | rerankerCohere                   | Reranks retrieved documents for relevance  | Kknowledge_base          | Kknowledge_base          |                                                                                              |
| AI Agent               | agent                            | Combines model, memory, and tools to reply | OpenAI Chat Model, Simple Memory, Kknowledge_base | Send message | ## Ai agent                                                                                  |
| Send message           | whatsApp                         | Sends AI-generated reply on WhatsApp       | AI Agent                 |                          |                                                                                              |
| Sticky Note1           | stickyNote                      | Documentation for AI Agent and WhatsApp flow |                         |                          |                                                                                              |
| Sticky Note2           | stickyNote                      | Full workflow overview and detailed docs  |                          |                          | See full workflow explanation, guardrails, setup, and troubleshooting                       |
| Sticky Note3           | stickyNote                      | Documentation for Google Drive ingestion   |                          |                          | See detailed notes on Google Drive → Supabase ingestion                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Document Ingestion Flow:**

   1. Add a **Google Drive Trigger** node:  
      - Set event to `fileUpdated`.  
      - Set folder to watch by folder ID (e.g., `1JCjixUoufxZRiFatfq-Nihr_vYRVoo2r`).  
      - Schedule polling weekly on Sunday (adjust as needed).  
      - Configure Google Drive OAuth2 credentials.

   2. Add a **Set (Edit Fields)** node:  
      - Assign `id` = `{{$json.id}}` to pass file ID downstream.

   3. Add a **Google Drive** node (Download file):  
      - Operation: `download`.  
      - File ID: `{{$json.id}}`.  
      - Enable Google Docs conversion to `text/plain`.  
      - Use same Google Drive OAuth2 credentials.

   4. Add **Character Text Splitter** (LangChain):  
      - Separator: `.`  
      - Chunk Size: 2000  
      - Chunk Overlap: 300

   5. Add **Default Data Loader**:  
      - Data Type: binary (file content)  
      - Text Splitting Mode: custom (using Character Text Splitter node)

   6. Add **Embeddings OpenAI** node:  
      - Use OpenAI API credentials.  
      - Use default or preferred embedding model.

   7. Add **Supabase Vector Store** node:  
      - Mode: `insert`  
      - Table name: `documents`  
      - Query name: `match_documents`  
      - Configure Supabase credentials (URL + API key).

   8. Connect nodes in order:  
      Google Drive Trigger → Edit Fields → Download file → Default Data Loader → Character Text Splitter → Embeddings OpenAI → Supabase Vector Store.

   9. (Optional) Add sticky note with documentation for future reference.

2. **Create WhatsApp AI Agent Interaction Flow:**

   1. Add **WhatsApp Trigger** node:  
      - Subscribe to `messages` update events.  
      - Set webhook and verify token.  
      - Configure WhatsApp API credentials (App ID, Secret, Verify Token).

   2. Add **Switch** node:  
      - Condition on `{{$json.messages[0].type}}`.  
      - Branch 1: equals `text` → direct to Merge.  
      - Branch 2: equals `audio` → HTTP Request.

   3. For audio branch:  
      - Add **HTTP Request** node:  
        - Method: GET  
        - URL: `https://graph.facebook.com/v21.0/{{$json.messages[0].audio.id}}`  
        - Query param: `access_token` with valid Facebook token.  
      - Add **HTTP Request1** node:  
        - Download file from URL returned by previous node.  
        - Header: `Authorization: Bearer <PAGE_ACCESS_TOKEN>`.  
      - Add **Translate a recording** node (OpenAI Whisper):  
        - Translates audio to text with OpenAI credentials.

   4. Add **Merge** node:  
      - Merge text branch and audio transcription branch.

   5. Add **Set (Edit Fields1)** node:  
      - Assign:  
        - `Phone` = `{{$json.messages[0].from}}`  
        - `text ` = concatenation of transcription text and typed text (`{{$json.text}}{{$json.messages[0].text.body}}`)

   6. Add **OpenAI Chat Model** node:  
      - Model: `gpt-4.1-mini`  
      - Temperature: 0.3  
      - Use OpenAI credentials.

   7. Add **Simple Memory** node:  
      - Session Key: `{{$json.Phone}}`  
      - Maintains user conversation context.

   8. Add **Kknowledge_base** node (Supabase vector store as retrieval tool):  
      - Mode: `retrieve-as-tool`  
      - Table: `documents`  
      - TopK: 10  
      - Use reranker: true (wired to Cohere)  
      - Tool description specific to clinic query handling.  
      - Use Supabase credentials.

   9. Add **Reranker Cohere** node:  
      - Use Cohere API credentials.  
      - Connect as reranker for Supabase query results.

   10. Add **AI Agent** node:  
       - Connect OpenAI Chat Model as language model.  
       - Connect Simple Memory as memory.  
       - Connect Kknowledge_base as tool.  
       - Configure system prompt with clinic-specific tone, guardrails, and behavior rules.  
       - On error: continue with regular output.

   11. Add **Send message** node (WhatsApp):  
       - Operation: `send`  
       - Text Body: `{{$json.output}}` from AI Agent  
       - Recipient Phone Number: `{{$('WhatsApp Trigger').item.json.messages[0].from}}`  
       - Phone Number ID and WhatsApp API credentials configured.

   12. Connect nodes in order:  
       WhatsApp Trigger → Switch → (text) Merge → Edit Fields1 → AI Agent → Send message  
       Switch → (audio) HTTP Request → HTTP Request1 → Translate a recording → Merge

   13. Add sticky notes documenting AI agent behavior, guardrails, and flow logic.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow turns incoming WhatsApp messages (text or audio) into short, friendly clinic replies using an AI Agent with memory and a Supabase-powered knowledge base. Guardrails enforce reply length ≤100 words, no pricing unless in KB, and no unsolicited appointment booking. The system prompt is tailored for HolistiCare clinic's professional and empathetic tone.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Section Sticky Note2: Full workflow overview and operational instructions                                                         |
| Auto-ingest Google Drive docs into a Supabase Vector Store for your knowledge base. The workflow watches a specific folder, downloads updated files, converts and chunks text, embeds with OpenAI, and inserts into Supabase for RAG. Supports Google Docs conversion to plain text. To keep chatbot responses fresh and grounded.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Section Sticky Note3: Google Drive ingestion instructions                                                                           |
| Guardrails baked into AI Agent include: replies under 100 words, concise and empathetic tone, no emojis, no unsolicited pricing info, no medical advice (recommend consult instead), no pushing appointments unless asked, and quick handoff to phone/WhatsApp if unclear.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | AI Agent system prompt and Sticky Note2                                                                                           |
| Credentials required: WhatsApp Trigger & Send nodes need Meta WhatsApp Business credentials; OpenAI API for chat and transcription; Supabase API for vector store; Google Drive OAuth2 for document ingestion; Cohere API for reranking. Ensure tokens are valid and refreshed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | General credential setup notes                                                                                                   |
| To test: send text and audio WhatsApp messages and verify AI replies; test queries for unavailable services or prices to verify guardrails; monitor logs for errors in media download or transcription; ensure Supabase table `documents` is populated with relevant, clean data.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Testing instructions from Sticky Note2                                                                                           |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.