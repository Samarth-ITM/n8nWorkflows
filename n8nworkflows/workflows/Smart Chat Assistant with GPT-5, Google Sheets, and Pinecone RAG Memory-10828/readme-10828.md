Smart Chat Assistant with GPT-5, Google Sheets, and Pinecone RAG Memory

https://n8nworkflows.xyz/workflows/smart-chat-assistant-with-gpt-5--google-sheets--and-pinecone-rag-memory-10828


# Smart Chat Assistant with GPT-5, Google Sheets, and Pinecone RAG Memory

### 1. Workflow Overview

This workflow implements a **Smart Chat Assistant** that leverages GPT-5, Google Sheets, and Pinecone for Retrieval-Augmented Generation (RAG) with memory and auto-logging of conversations. It is designed to provide context-aware AI responses enriched by both structured data (Google Sheets) and semantic knowledge (Pinecone vector store). The workflow also supports:

- Auto-indexing of new knowledge documents added via Google Drive into Pinecone.
- Weekly email exports of conversation logs.
- Session-based short-term memory to maintain conversational state.
- Automated logging of every chat interaction for audit and analytics.

**Logical blocks:**

- **1.1 Input Reception & Pre-Processing:** Receives chat messages via webhook, enriches with metadata, and prepares data for AI processing.
- **1.2 AI Processing Core:** Invokes GPT-5 with enriched context from Google Sheets and Pinecone, using enforced JSON output and session memory.
- **1.3 Conversation Logging:** Logs chat interactions to Google Sheets.
- **1.4 Knowledge Base Auto-Indexing:** Watches a Google Drive folder for new files, processes and indexes them into Pinecone.
- **1.5 Weekly Conversation History Export:** Scheduled aggregation and emailing of chat logs as attachments.
- **1.6 Credentials & Security Reminder:** Notes about required API credentials and security best practices.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Pre-Processing

**Overview:**  
This block captures incoming user chat messages via a webhook, enriches them with intent, topic, and parameters, and formats them for the AI agent.

**Nodes Involved:**  
- When chat message received  
- Format Data For AI Agent

##### Node: When chat message received  
- **Type:** Chat Trigger (langchain)  
- **Role:** Entry point capturing incoming chat messages via webhook.  
- **Config:** Uses a webhook ID to receive chat input. No specific options set.  
- **Input:** External HTTP chat message.  
- **Output:** JSON with user message content.  
- **Edge cases:** Missing or malformed webhook payload, network timeouts.

##### Node: Format Data For AI Agent  
- **Type:** Set node  
- **Role:** Adds metadata fields to incoming chat JSON: intent ("Chat"), topic ("AI Seo Basics"), content_id, and additional parameters. Prepares enriched data for AI input.  
- **Config:** Static values for intent and topic; content_id set to "C001"; parameter field copies from input JSON.  
- **Input:** From chat trigger node.  
- **Output:** JSON enriched with intent, topic, content_id, parameter.  
- **Edge cases:** Missing parameters field could cause empty parameter object.

---

#### 1.2 AI Processing Core

**Overview:**  
This core block uses GPT-5 with memory and dual data retrieval: structured context from Google Sheets and semantic knowledge from Pinecone. It enforces JSON output for consistent responses.

**Nodes Involved:**  
- Short-Term Memory  
- OpenAI Chat Model  
- AI Agent (Chat Composer)  
- Output Parser (JSON Enforcement)  
- Get Previous Content from Sheet  
- Pinecone Vector Store Query for Knowledge Base  
- Embeddings OpenAI Query for Chat modal

##### Node: Short-Term Memory  
- **Type:** Langchain memory buffer window  
- **Role:** Maintains a sliding window of last 7 conversational turns keyed by a custom session key.  
- **Config:** sessionKey "chat-rag-session", contextWindowLength 7.  
- **Input:** From AI Chat Model output.  
- **Output:** Provides memory context to AI Agent node.  
- **Edge cases:** Session key collisions or missing sessions.

##### Node: OpenAI Chat Model  
- **Type:** Langchain OpenAI Chat Model  
- **Role:** Executes GPT-5 calls for language model responses.  
- **Config:** Model set dynamically via expression (likely "gpt-5"), no special options. Credentials use OpenAI API key.  
- **Input:** Receives prompt from AI Agent node.  
- **Output:** Raw language model output text.  
- **Edge cases:** API key issues, rate limits, timeouts.

##### Node: AI Agent (Chat Composer)  
- **Type:** Langchain Agent  
- **Role:** Core AI agent taking user message, intent, topic, and fetched context to compose a chat response.  
- **Config:**  
  - System message defines behavior: professional, concise, uses context from Google Sheets or Pinecone, handles greetings, SEO/content queries, and missing context gracefully.  
  - Output enforced as concise JSON with "reply" and "context_used" fields.  
- **Input:** Enriched user message + context + memory.  
- **Output:** Structured JSON response with reply text and used context.  
- **Edge cases:** Missing context, malformed input, response formatting errors.

##### Node: Output Parser (JSON Enforcement)  
- **Type:** Langchain Output Parser Structured  
- **Role:** Parses AI output to enforce JSON schema `{reply: string, context_used: [string]}`  
- **Config:** JSON schema example provided for strict validation.  
- **Input:** Raw AI Agent output.  
- **Output:** Validated JSON object.  
- **Edge cases:** Parsing errors if AI outputs non-conforming text.

##### Node: Get Previous Content from Sheet  
- **Type:** Google Sheets Tool  
- **Role:** Retrieves structured conversation or context data from Google Sheets to enrich AI input.  
- **Config:** Document ID and sheet name configured with OAuth2 credentials.  
- **Input:** Triggered as a tool by AI Agent node to fetch context.  
- **Output:** Context data from Sheets.  
- **Edge cases:** Auth errors, sheet not found, empty content.

##### Node: Pinecone Vector Store Query for Knowledge Base  
- **Type:** Langchain Vector Store Pinecone (retrieve-as-tool)  
- **Role:** Retrieves top 5 relevant vectors from Pinecone knowledge base for context.  
- **Config:** Index "whatsappchatbot" specified; tool description instructs AI to use knowledge base as source of truth.  
- **Input:** Query embedding from AI Agent embedding node.  
- **Output:** Retrieved semantic context.  
- **Edge cases:** API key issues, index not found, query failures.

##### Node: Embeddings OpenAI Query for Chat modal  
- **Type:** Langchain Embeddings OpenAI  
- **Role:** Generates embeddings for user queries to query Pinecone.  
- **Config:** 512 dimensions, OpenAI API credentials.  
- **Input:** User message text.  
- **Output:** Vector embeddings.  
- **Edge cases:** API issues, malformed input.

---

#### 1.3 Conversation Logging

**Overview:**  
Logs every chat interaction to Google Sheets, capturing timestamp, user input, detected intent, topic, and AI response for analytics and audit.

**Nodes Involved:**  
- Conversation Logging

##### Node: Conversation Logging  
- **Type:** Google Sheets (appendOrUpdate)  
- **Role:** Appends chat interaction data to a specified Google Sheet document and sheet name.  
- **Config:** Document ID and sheet name configured with OAuth2 credentials.  
- **Input:** JSON from AI Agent (final response with metadata).  
- **Output:** Confirmation of append/update.  
- **Edge cases:** Sheet permissions, quota limits, connectivity errors.

---

#### 1.4 Knowledge Base Auto-Indexing

**Overview:**  
Watches a Google Drive folder for new files (PDFs, Docs), downloads them, splits text into chunks, generates embeddings, and inserts them into Pinecone for RAG retrieval.

**Nodes Involved:**  
- Google Drive Trigger  
- Download file  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Pinecone Vector Store Insert

##### Node: Google Drive Trigger  
- **Type:** Google Drive Trigger  
- **Role:** Watches a specific Google Drive folder for newly created files every minute.  
- **Config:** Folder ID specified, polling mode every minute. OAuth2 for Google Drive.  
- **Input:** New file event.  
- **Output:** File metadata JSON.  
- **Edge cases:** Permission issues, polling delays.

##### Node: Download file  
- **Type:** Google Drive  
- **Role:** Downloads the detected new file from Google Drive.  
- **Config:** File ID from trigger node.  
- **Input:** File metadata.  
- **Output:** Binary file data.  
- **Edge cases:** File access denied, network issues.

##### Node: Recursive Character Text Splitter  
- **Type:** Langchain Text Splitter  
- **Role:** Splits downloaded file text into chunks of 500 chars with 200 overlap for embedding.  
- **Config:** chunkSize=500, chunkOverlap=200.  
- **Input:** Text from Default Data Loader.  
- **Output:** Array of text chunks.  
- **Edge cases:** Empty files, splitting errors.

##### Node: Default Data Loader  
- **Type:** Langchain Document Default Data Loader  
- **Role:** Converts binary file into text and attaches metadata (file name).  
- **Config:** Text splitting mode "custom", metadata includes file name.  
- **Input:** Binary data from Download file.  
- **Output:** Text content for splitting.  
- **Edge cases:** Unsupported file types.

##### Node: Embeddings OpenAI  
- **Type:** Langchain Embeddings OpenAI  
- **Role:** Generates embeddings for text chunks.  
- **Config:** 512 dimensions, OpenAI API credentials.  
- **Input:** Text chunks.  
- **Output:** Embeddings for Pinecone insertion.  
- **Edge cases:** API rate limits.

##### Node: Pinecone Vector Store Insert  
- **Type:** Langchain Vector Store Pinecone (insert mode)  
- **Role:** Inserts embeddings into the Pinecone index for future retrieval.  
- **Config:** Pinecone index ID configured, batch size 1000.  
- **Input:** Embeddings from OpenAI node.  
- **Output:** Confirmation of insertion.  
- **Edge cases:** Index quota, API errors.

---

#### 1.5 Weekly Conversation History Export

**Overview:**  
Runs every Monday at 10 PM, aggregates all chat logs from Google Sheets, converts them to a text file, and emails it as an attachment via Gmail.

**Nodes Involved:**  
- Schedule Trigger  
- Get Data from Sheet.  
- Aggregate  
- Convert Data to file  
- Check File Exist  
- Send Chat History with attachment

##### Node: Schedule Trigger  
- **Type:** Schedule Trigger  
- **Role:** Triggers the export workflow every Monday at 22:00.  
- **Config:** weekly interval, day=Monday, hour=22.  
- **Input:** none (time-based).  
- **Output:** Trigger signal.

##### Node: Get Data from Sheet.  
- **Type:** Google Sheets  
- **Role:** Retrieves all chat log entries from the configured Google Sheet.  
- **Config:** Document ID and sheet name set with OAuth2.  
- **Input:** Trigger from Schedule node.  
- **Output:** Array of logged chat data.  
- **Edge cases:** Empty sheet, access issues.

##### Node: Aggregate  
- **Type:** Aggregate  
- **Role:** Combines all rows from Google Sheets into a single dataset for export.  
- **Config:** Aggregate all item data.  
- **Input:** Data array.  
- **Output:** Single aggregated dataset.

##### Node: Convert Data to file  
- **Type:** Convert To File  
- **Role:** Converts aggregated data into a text file (likely CSV or plain text).  
- **Config:** Operation "toText", source property "data".  
- **Input:** Aggregated dataset.  
- **Output:** Binary text file.

##### Node: Check File Exist  
- **Type:** If node  
- **Role:** Validates that the converted file's MIME type is application/json (likely for error checking or format validation).  
- **Config:** Condition: binary.data.mimeType equals application/json.  
- **Input:** File binary data.  
- **Output:** True branch leads to sending email.  
- **Edge cases:** MIME type mismatch; file conversion failure.

##### Node: Send Chat History with attachment  
- **Type:** Gmail node  
- **Role:** Sends the chat history file as an email attachment to a recipient.  
- **Config:**  
  - Subject: "chat_history_DD" where DD is current day.  
  - Message body with greeting and instructions.  
  - Attachment is the converted file binary.  
  - OAuth2 Gmail credentials used.  
- **Input:** File binary from Check File Exist node.  
- **Output:** Email sent confirmation.  
- **Edge cases:** Email send failures, attachment size limits.

---

#### 1.6 Credentials & Security Reminder

**Overview:**  
Notes to users about the required credentials and security best practices.

**Nodes Involved:**  
- Sticky Note6

##### Node: Sticky Note6  
- **Type:** Sticky Note  
- **Role:** Contains instructions to use OAuth2 for Google services and API keys for OpenAI and Pinecone. Warns to replace IDs and never share keys publicly.  
- **Config:** Static content.  
- **Input/Output:** None.  
- **Edge cases:** None.

---

### 3. Summary Table

| Node Name                                | Node Type                                 | Functional Role                                | Input Node(s)                       | Output Node(s)                        | Sticky Note                                                                                                  |
|-----------------------------------------|-------------------------------------------|------------------------------------------------|-----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note                             | Sticky Note                              | Workflow overview and setup instructions       | -                                 | -                                   | ## 🤖 AI Chat Assistant with RAG & Auto-Logging ... See Setup steps                                            |
| Sticky Note1                            | Sticky Note                              | Chat interface & pre-processing description     | -                                 | -                                   | ## 💬 Chat Interface & Pre-Processing ...                                                                    |
| Sticky Note2                            | Sticky Note                              | AI agent core description                        | -                                 | -                                   | ## 🧠 AI Agent Core ...                                                                                        |
| Sticky Note3                            | Sticky Note                              | Conversation logging description                 | -                                 | -                                   | ## 📊 Conversation Logging ...                                                                                 |
| Sticky Note4                            | Sticky Note                              | Knowledge base auto-indexing description         | -                                 | -                                   | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Sticky Note5                            | Sticky Note                              | Weekly chat history export description           | -                                 | -                                   | ## 📧 Weekly Chat History Export ...                                                                            |
| Sticky Note6                            | Sticky Note                              | Credentials & security reminder                   | -                                 | -                                   | ## 🔐 Credentials & Security ...                                                                                |
| When chat message received              | Chat Trigger (langchain)                 | Entry point to receive user chat messages        | -                                 | Format Data For AI Agent            | ## 💬 Chat Interface & Pre-Processing ...                                                                    |
| Format Data For AI Agent                | Set                                     | Enrich chat data with intent and topic metadata | When chat message received        | AI Agent (Chat Composer)            | ## 💬 Chat Interface & Pre-Processing ...                                                                    |
| Short-Term Memory                       | Langchain Memory Buffer Window           | Maintains short-term conversational memory       | OpenAI Chat Model                 | AI Agent (Chat Composer)            | ## 🧠 AI Agent Core ...                                                                                        |
| OpenAI Chat Model                      | Langchain OpenAI Chat Model               | Calls GPT-5 for language generation               | AI Agent (Chat Composer)          | AI Agent (Chat Composer)            | ## 🧠 AI Agent Core ...                                                                                        |
| AI Agent (Chat Composer)                | Langchain Agent                          | Core AI logic combining input, memory, context   | Format Data For AI Agent, Short-Term Memory, OpenAI Chat Model, Get Previous Content from Sheet, Pinecone Vector Store Query for Knowledge Base, Output Parser | Conversation Logging               | ## 🧠 AI Agent Core ...                                                                                        |
| Output Parser (JSON Enforcement)        | Langchain Output Parser Structured       | Enforce JSON output format from AI                 | AI Agent (Chat Composer)          | AI Agent (Chat Composer)            | ## 🧠 AI Agent Core ...                                                                                        |
| Get Previous Content from Sheet         | Google Sheets Tool                       | Retrieves structured context from Google Sheets   | -                               | AI Agent (Chat Composer)            | ## 🧠 AI Agent Core ...                                                                                        |
| Pinecone Vector Store Query for Knowledge Base | Langchain Vector Store Pinecone (retrieve-as-tool) | Retrieves semantic knowledge base context          | Embeddings OpenAI Query for Chat modal | AI Agent (Chat Composer)            | ## 🧠 AI Agent Core ...                                                                                        |
| Embeddings OpenAI Query for Chat modal | Langchain Embeddings OpenAI             | Generates embeddings to query Pinecone            | AI Agent (Chat Composer)          | Pinecone Vector Store Query for Knowledge Base | ## 🧠 AI Agent Core ...                                                                                        |
| Conversation Logging                   | Google Sheets                            | Logs chat interactions for analytics and audit    | AI Agent (Chat Composer)          | -                                 | ## 📊 Conversation Logging ...                                                                                 |
| Google Drive Trigger                   | Google Drive Trigger                     | Watches Google Drive folder for new files          | -                               | Download file                      | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Download file                         | Google Drive                            | Downloads new files detected by trigger            | Google Drive Trigger             | Pinecone Vector Store Insert        | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Default Data Loader                   | Langchain Document Default Data Loader  | Converts binary files to text with metadata        | Recursive Character Text Splitter | Pinecone Vector Store Insert        | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Recursive Character Text Splitter      | Langchain Text Splitter                 | Splits text into chunks for embedding generation   | Default Data Loader              | Default Data Loader                 | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Embeddings OpenAI                    | Langchain Embeddings OpenAI             | Generates embeddings for document chunks           | Default Data Loader              | Pinecone Vector Store Insert        | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Pinecone Vector Store Insert           | Langchain Vector Store Pinecone (insert) | Inserts document embeddings into Pinecone index     | Download file, Embeddings OpenAI | -                                 | ## 📂 Knowledge Base Auto-Indexing ...                                                                         |
| Schedule Trigger                     | Schedule Trigger                       | Triggers weekly export of chat history             | -                               | Get Data from Sheet.                | ## 📧 Weekly Chat History Export ...                                                                            |
| Get Data from Sheet.                 | Google Sheets                         | Retrieves chat logs from Google Sheets              | Schedule Trigger                | Aggregate                         | ## 📧 Weekly Chat History Export ...                                                                            |
| Aggregate                           | Aggregate                            | Aggregates chat log rows into single dataset         | Get Data from Sheet.            | Convert Data to file               | ## 📧 Weekly Chat History Export ...                                                                            |
| Convert Data to file                | Convert To File                      | Converts aggregated data to text file                 | Aggregate                      | Check File Exist                  | ## 📧 Weekly Chat History Export ...                                                                            |
| Check File Exist                   | If                                  | Validates file MIME type before emailing             | Convert Data to file            | Send Chat History with attachment | ## 📧 Weekly Chat History Export ...                                                                            |
| Send Chat History with attachment | Gmail                               | Sends chat history file by email                      | Check File Exist               | -                                 | ## 📧 Weekly Chat History Export ...                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Chat Trigger node:**
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Set a webhook ID (auto-generated).
   - No special options.
   - This node is the chat entry point.

3. **Add a Set node ("Format Data For AI Agent"):**
   - Type: `Set`
   - Add fields:
     - `intent`: string, value `"Chat"`
     - `topic`: string, value `"AI Seo Basics"`
     - `content_id`: string, value `"C001"`
     - `parameter`: object, value copied from incoming JSON `parameter`
   - Connect output of Chat Trigger to this node.

4. **Add the AI Agent block:**

   a. **Add Langchain OpenAI Chat Model node:**
      - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
      - Model: expression to select GPT-5 (e.g. `"gpt-5"`)
      - Use OpenAI API credentials.
      - Connect later to AI Agent node as language model.

   b. **Add Langchain Memory Buffer Window node ("Short-Term Memory"):**
      - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
      - sessionKey: `"chat-rag-session"`
      - sessionIdType: `"customKey"`
      - contextWindowLength: `7`
      - Connect output of OpenAI Chat Model to this node.

   c. **Add Google Sheets Tool node ("Get Previous Content from Sheet"):**
      - Type: `Google Sheets Tool`
      - Configure Document ID and Sheet name with OAuth2.
      - Used as AI Agent tool to fetch structured context.

   d. **Add Embeddings OpenAI node for query:**
      - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`
      - Dimensions: `512`
      - Use OpenAI API credentials.

   e. **Add Pinecone Vector Store Query node:**
      - Type: `@n8n/n8n-nodes-langchain.vectorStorePinecone`
      - Mode: `retrieve-as-tool`
      - TopK: 5
      - Pinecone index: `"whatsappchatbot"` or your index
      - Use Pinecone API credentials.
      - Connect output of Embeddings OpenAI query to this node.

   f. **Add Output Parser (JSON Enforcement):**
      - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
      - Provide JSON schema example:
        ```
        {
          "reply": "string",
          "context_used": ["string"]
        }
        ```
      - Connect output of AI Agent node to this parser.

   g. **Add AI Agent node ("AI Agent (Chat Composer)"):**
      - Type: `@n8n/n8n-nodes-langchain.agent`
      - Configure prompt using expressions to include user message, intent, topic, context (from Sheets or Pinecone), and memory.
      - System message defines behavior and response formatting rules.
      - Set output parser to the Output Parser node.
      - Add tools: Google Sheets Tool and Pinecone Vector Store Query.
      - Connect inputs:
        - User message: from "Format Data For AI Agent"
        - Memory: from Short-Term Memory node
        - Language Model: from OpenAI Chat Model
        - Tools: Google Sheets Tool, Pinecone Vector Store Query
        - Output Parser: Output Parser node
      - Output connects to Conversation Logging node.

5. **Add Google Sheets node ("Conversation Logging"):**
   - Type: `Google Sheets`
   - Operation: `appendOrUpdate`
   - Configure Document ID and Sheet name with OAuth2.
   - Connect output of AI Agent node to this node.

6. **Knowledge Base Auto-Indexing:**

   a. Add Google Drive Trigger node:
      - Type: `Google Drive Trigger`
      - Event: `fileCreated`
      - Poll every minute
      - Watch specific folder by folder ID with OAuth2.

   b. Add Google Drive node ("Download file"):
      - Type: `Google Drive`
      - Operation: `download`
      - File ID: expression from trigger node.

   c. Add Langchain Document Default Data Loader:
      - Type: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`
      - DataType: `binary`
      - Text Splitting Mode: `custom`
      - Add metadata: file name from downloaded file.

   d. Add Langchain Recursive Character Text Splitter:
      - Type: `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`
      - Chunk Size: 500
      - Chunk Overlap: 200

   e. Add Langchain Embeddings OpenAI node:
      - Dimensions: 512
      - Use OpenAI API credentials.

   f. Add Pinecone Vector Store Insert node:
      - Mode: `insert`
      - Pinecone index ID configured.
      - Embedding batch size: 1000
      - Use Pinecone API credentials.

   - Connect nodes in order: Google Drive Trigger → Download file → Recursive Character Text Splitter → Default Data Loader → Embeddings OpenAI → Pinecone Vector Store Insert

7. **Weekly Chat History Export:**

   a. Add Schedule Trigger node:
      - Interval: weekly
      - Trigger at Monday 22:00

   b. Add Google Sheets node ("Get Data from Sheet."):
      - Retrieve all chat logs.
      - Configured with OAuth2.

   c. Add Aggregate node:
      - Operation: aggregate all item data.

   d. Add Convert To File node:
      - Operation: toText
      - Source Property: `data`

   e. Add If node ("Check File Exist"):
      - Condition: `binary.data.mimeType` equals `application/json`

   f. Add Gmail node ("Send Chat History with attachment"):
      - Subject: `chat_history_DD` (DD = current day)
      - Email body with greeting and attachment
      - Use Gmail OAuth2 credentials
      - Attach file from Convert To File node.

   - Connect nodes: Schedule Trigger → Get Data from Sheet → Aggregate → Convert Data to File → Check File Exist → Send Chat History with attachment

8. **Add Sticky Notes for documentation and reminders as desired.**

9. **Configure all credentials:**
   - OpenAI API key
   - Google Sheets OAuth2
   - Google Drive OAuth2
   - Pinecone API key
   - Gmail OAuth2

10. **Replace all document IDs, folder IDs, and index names with your own resources.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow integrates GPT-5 with Google Sheets and Pinecone for advanced Retrieval-Augmented Generation and conversational memory recording. It includes auto-indexing knowledge from Google Drive files and weekly email export of chat logs.                                                                                                                                    | Overview from Sticky Note 1                                                                       |
| Ensure all API keys and OAuth2 credentials are securely stored and never shared publicly to avoid unauthorized access.                                                                                                                                                                                                                                                           | Sticky Note6 about Credentials & Security                                                       |
| Setup requires configuring Google Sheets document IDs, folder IDs for Google Drive, and Pinecone vector index names.                                                                                                                                                                                                                                                            | Sticky Note setup instructions                                                                   |
| Use the chat webhook URL from the "When chat message received" node to send test messages and debug.                                                                                                                                                                                                                                                                              | Sticky Note setup instructions                                                                   |
| For knowledge base management, upload PDFs or docs to the monitored Google Drive folder to auto-index them into Pinecone for RAG.                                                                                                                                                                                                                                               | Sticky Note4                                                                                     |
| Weekly chat history emails are sent every Monday at 10 PM with the previous chat logs attached to an email via Gmail OAuth2.                                                                                                                                                                                                                                                    | Sticky Note5                                                                                     |
| For advanced prompt engineering, the AI Agent system message defines professional, concise, and context-aware behavior, including handling greetings, SEO queries, and confidence in responses.                                                                                                                                                                                   | AI Agent (Chat Composer) system message configuration                                           |
| [n8n Documentation](https://docs.n8n.io) and [OpenAI API Reference](https://platform.openai.com/docs) are useful references for node configuration and API usages.                                                                                                                                                                                                             | General resources                                                                                |

---

*Disclaimer: The provided text is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.*