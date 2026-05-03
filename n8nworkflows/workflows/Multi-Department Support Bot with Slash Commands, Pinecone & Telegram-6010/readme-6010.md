Multi-Department Support Bot with Slash Commands, Pinecone & Telegram

https://n8nworkflows.xyz/workflows/multi-department-support-bot-with-slash-commands--pinecone---telegram-6010


# Multi-Department Support Bot with Slash Commands, Pinecone & Telegram

### 1. Workflow Overview

This workflow implements a **Multi-Department Support Bot** for Telegram, integrating slash command triggers, document ingestion from Google Drive, vector storage with Pinecone, and AI-powered responses via LangChain using Cohere embeddings and OpenRouter language models. It targets customer support scenarios where users interact through Telegram commands to access different department knowledge bases (Billing, Return Policy, Tech Support).

The workflow is logically divided into three main blocks:

- **1.1 Telegram Interaction & Session Management:** Handles incoming Telegram messages, interprets slash commands, manages user session state and department assignment stored in a PostgreSQL database, and routes users accordingly.

- **1.2 AI Query Processing & Response:** For active user sessions with a selected department, routes user queries to a LangChain AI agent configured with tools linked to department-specific Pinecone vector stores, generating context-aware answers.

- **1.3 Document Ingestion & Vector Store Updating:** Watches designated Google Drive folders for new PDFs related to each department, downloads and processes them with text splitting and Cohere embeddings, then inserts embeddings into corresponding Pinecone namespaces to keep the knowledge base up-to-date.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Interaction & Session Management

**Overview:**  
This block receives Telegram messages, identifies commands (/start, /end, department commands), initializes user sessions in PostgreSQL, updates user active state and department, and sends appropriate Telegram replies.

**Nodes Involved:**  
- Telegram Trigger  
- Execute a SQL query1 (Insert user if new)  
- Select rows from a table (Get user session)  
- Switch1 (Decides next step based on message text and session status)  
- Send a text message (Welcome prompt)  
- Send a text message4 (Instructions for /start and /end)  
- AI Agent3 (Forward to AI if active session)  
- Execute a SQL query2 (Deactivate session on /end)  
- Switch (Route based on department command)  
- Return policy, talk technical, billing (Telegram messages confirming department choice)  
- Execute SQL queries (return policy1, tech questions, billing1) (Update session department and active status)  
- Send a text message3 (Prompt for correct department if unknown)  
- Execute a SQL query3 (Reset active state and department on /start)  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Entry point for all Telegram updates (listening to messages)  
  - Config: Listens to "message" updates only  
  - Output: Raw Telegram message JSON including chat and user data  
  - Edge cases: Missing message text, malformed payload, Telegram API downtime  

- **Execute a SQL query1**  
  - Type: PostgreSQL Execute Query  
  - Role: Inserts user into `tg_user_sessions` if not exists, initializing with inactive state  
  - Query uses Telegram user ID from incoming message  
  - Edge cases: Database connection failure, query syntax error  

- **Select rows from a table**  
  - Type: PostgreSQL Select Query  
  - Role: Retrieves user session row by user_id to check active status and department  
  - Output feeds condition switch  
  - Edge cases: No rows returned (new user), DB failure  

- **Switch1**  
  - Type: Switch node with expression mode  
  - Role: Determines workflow path based on message text and session active flag  
  - Logic:  
    - "/start" → path 0 (start welcome message)  
    - "/end" → path 3 (deactivate session)  
    - If active=true and department assigned → path 2 (send query to AI)  
    - If active=true but no department → path 4 (fallback switch for commands)  
    - Otherwise path 1 (prompt to start)  
  - Edge cases: Text casing, unexpected input  

- **Send a text message**  
  - Type: Telegram node  
  - Role: Sends greeting and instructions on /start  
  - Uses chat id from Telegram Trigger  
  - Connected after Switch1 path 0  

- **Send a text message4**  
  - Type: Telegram node  
  - Role: Sends instruction to use /start or /end commands  
  - Connected after Switch1 path 1  

- **AI Agent3**  
  - Type: LangChain AI Agent node  
  - Role: Processes user queries when session is active with department  
  - Uses Telegram message text as input prompt  
  - Configured with system message guiding agent to respect department boundaries  
  - Connected after Switch1 path 2  

- **Execute a SQL query2**  
  - Type: PostgreSQL Execute Query  
  - Role: Deactivates user session and clears department on /end command  
  - Connected after Switch1 path 3  

- **Switch**  
  - Type: Switch node  
  - Role: Routes user commands to department handlers based on message text  
  - Conditions for "/ReturnPolicy", "/TechSupport", "/billing"  
  - Fallback sends prompt for valid department  
  - Connected after Switch1 path 4  

- **return policy, talk technical, billing**  
  - Type: Telegram message nodes  
  - Role: Confirm department choice to user  
  - Each sends a simple textual reply like "lets talk return policy"  
  - Connected from Switch node outputs  

- **return policy1, tech questions, billing1**  
  - Type: PostgreSQL Execute Query  
  - Role: Update session row setting active=true and department accordingly  
  - Use chat id from Telegram message to identify user row  

- **Send a text message3**  
  - Type: Telegram node  
  - Role: Prompt user to provide correct department command if unrecognized input  

- **Execute a SQL query3**  
  - Type: PostgreSQL Execute Query  
  - Role: Resets active flag and clears department on /start command (clears previous session if any)  

**Potential Edge Cases and Failure Points:**  
- Missing or malformed Telegram message text or user data  
- SQL query failures or connection issues  
- User entering unrecognized commands  
- Concurrent updates to user session causing race conditions  

---

#### 2.2 AI Query Processing & Response

**Overview:**  
This block handles user queries routed after session and department determination. It uses LangChain’s AI Agent with integrated Pinecone vector stores as knowledge tools for each department. The agent is memory-enabled and uses Cohere embeddings plus OpenRouter chat model.

**Nodes Involved:**  
- AI Agent3 (LangChain agent processing user input)  
- OpenRouter Chat Model3 (Language model used by agent)  
- Simple Memory3 (Session memory for conversation context)  
- Pinecone Vector Store6 (Return Policy vector tool)  
- Pinecone Vector Store7 (Tech Support vector tool)  
- Pinecone Vector Store8 (Billing vector tool)  
- Embeddings Cohere6, 7, 8 (Embeddings for vector stores)  
- Send a text message1 (Sends AI agent output back to user)  

**Node Details:**

- **AI Agent3**  
  - Type: LangChain Agent with tools  
  - Role: Acts as main AI assistant responding to user queries  
  - Input: Telegram message text  
  - System prompt instructs respecting department context, declining out-of-scope questions, sign-off style  
  - Uses Pinecone vector stores as retrieval tools depending on department  
  - Integrated with conversation memory buffer (Simple Memory3) keyed by user ID for session continuity  
  - Edge cases: Model API failures, incorrect tool routing, memory desynchronization  

- **OpenRouter Chat Model3**  
  - Type: Language Model node (OpenRouter)  
  - Role: Provides LLM capabilities to AI Agent  
  - Configured with model "deepseek/deepseek-chat-v3-0324:free"  
  - Edge cases: API limits, latency, unexpected model output  

- **Simple Memory3**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores recent conversation history keyed by user_id for context retention  
  - Ensures AI agent responses consider prior dialog  
  - Edge cases: Memory overflow, improper session key  

- **Pinecone Vector Store6,7,8**  
  - Type: LangChain Pinecone Vector Store nodes  
  - Role: Provide retrieval tools for AI agent to fetch relevant knowledge from indexed documents  
  - Namespaces:  
    - Return Policy → "return policy"  
    - Tech Support → "tech ques"  
    - Billing → "billing"  
  - Each configured for "retrieve-as-tool" mode for LangChain agent tool integration  
  - Edge cases: Pinecone API outages, empty namespaces, indexing delays  

- **Embeddings Cohere6,7,8**  
  - Type: LangChain Embeddings using Cohere  
  - Role: Generate embeddings for documents to be stored in Pinecone vector stores  
  - Used in ingestion flows feeding vector stores and as embedding provider for queries  

- **Send a text message1**  
  - Type: Telegram node  
  - Role: Sends AI-generated answer text back to user chat  
  - Uses Telegram's from.id as chatId  
  - Edge cases: Telegram API failures, message formatting issues  

---

#### 2.3 Document Ingestion & Vector Store Updating

**Overview:**  
This block automates the ingestion of new documents uploaded to Google Drive folders for each department. When a new file is created, it downloads, processes, splits text, embeds with Cohere, and inserts vectors into the corresponding Pinecone namespace.

**Nodes Involved:**  
- Google Drive Trigger, Google Drive Trigger1, Google Drive Trigger2 (One per department folder)  
- Download file, Download file1, Download file2 (Download the newly added file)  
- Default Data Loader, Default Data Loader1, Default Data Loader2 (Load document content for processing)  
- Character Text Splitter, Character Text Splitter1, Character Text Splitter2 (Split long documents into manageable chunks)  
- Embeddings Cohere3, Embeddings Cohere4, Embeddings Cohere5 (Generate embeddings for chunks)  
- Pinecone Vector Store3, Pinecone Vector Store4, Pinecone Vector Store5 (Insert embeddings into Pinecone vector stores)  
- Sticky Note, Sticky Note1, Sticky Note2 (Annotations marking department ingestion flows)  

**Node Details:**

- **Google Drive Trigger / Trigger1 / Trigger2**  
  - Type: Google Drive Trigger nodes  
  - Role: Poll respective Google Drive folders monthly, trigger on new file creation  
  - Folder IDs correspond to department-specific folders:  
    - Billing folder  
    - Technical questions folder  
    - Return policy folder  
  - Edge cases: Drive permission errors, missed triggers due to polling intervals  

- **Download file / Download file1 / Download file2**  
  - Type: Google Drive node  
  - Role: Download newly created file in binary format for processing  
  - Input: fileId from trigger output  
  - Edge cases: File access revoked, download failures  

- **Default Data Loader / Default Data Loader1 / Default Data Loader2**  
  - Type: LangChain Document Loader  
  - Role: Load downloaded binary data as documents for embedding  
  - Text splitting mode: Custom (to handle large documents)  
  - Edge cases: Unsupported file types, corrupted files  

- **Character Text Splitter / Character Text Splitter1 / Character Text Splitter2**  
  - Type: LangChain Text Splitter  
  - Role: Splits document text into smaller chunks for embedding  
  - Edge cases: Over-splitting, chunk overlap misconfiguration  

- **Embeddings Cohere3 / 4 / 5**  
  - Type: LangChain Embeddings node using Cohere  
  - Role: Converts text chunks into vector embeddings for Pinecone  
  - Edge cases: API rate limits, embedding failures  

- **Pinecone Vector Store3 / 4 / 5**  
  - Type: LangChain Pinecone Vector Store node  
  - Role: Inserts embeddings into Pinecone index with department-specific namespaces  
  - Namespace mapping:  
    - Billing → "billing"  
    - Tech Support → "tech ques"  
    - Return Policy → "return policy"  
  - Edge cases: Pinecone insertion errors, namespace conflicts  

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                           | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                      |
|-------------------------|----------------------------------------|-----------------------------------------|----------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                       | Entry point for Telegram messages       |                                  | Execute a SQL query1                  |                                                                                                |
| Execute a SQL query1    | PostgreSQL Execute Query               | Insert user if new                       | Telegram Trigger                 | Select rows from a table              |                                                                                                |
| Select rows from a table| PostgreSQL Select Query                | Retrieve user session                    | Execute a SQL query1             | Switch1                             |                                                                                                |
| Switch1                | Switch                                | Route based on message and session      | Select rows from a table         | Send a text message, Send a text message4, AI Agent3, Execute a SQL query2, Switch |                                                                                                |
| Send a text message     | Telegram                              | Welcome message on /start                | Switch1                         | Execute a SQL query3                  |                                                                                                |
| Execute a SQL query3    | PostgreSQL Execute Query               | Reset active session on /start           | Send a text message              | Switch                              |                                                                                                |
| Send a text message4    | Telegram                              | Instruction message to use /start or /end| Switch1                         |                                    |                                                                                                |
| AI Agent3              | LangChain AI Agent                    | Process active user queries              | Switch1                         | Send a text message1                  |                                                                                                |
| Send a text message1    | Telegram                              | Send AI response to user                 | AI Agent3                      |                                    |                                                                                                |
| Execute a SQL query2    | PostgreSQL Execute Query               | Deactivate session on /end                | Switch1                         | Send a text message2                  |                                                                                                |
| Send a text message2    | Telegram                              | Confirm session ended message            | Execute a SQL query2             |                                    |                                                                                                |
| Switch                 | Switch                                | Route to department handlers             | Switch1                         | return policy, talk technical, billing, Send a text message3 |                                                                                                |
| return policy          | Telegram                              | Confirm Return Policy department chosen  | Switch                         | return policy1                      |                                                                                                |
| return policy1         | PostgreSQL Execute Query               | Set session active and department to Return Policy | return policy              |                                    |                                                                                                |
| talk technical         | Telegram                              | Confirm Technical Support department     | Switch                         | tech questions                     |                                                                                                |
| tech questions         | PostgreSQL Execute Query               | Set session active and department to Tech Support | talk technical             |                                    |                                                                                                |
| billing                | Telegram                              | Confirm Billing department chosen        | Switch                         | billing1                          |                                                                                                |
| billing1               | PostgreSQL Execute Query               | Set session active and department to Billing | billing                    |                                    |                                                                                                |
| Send a text message3    | Telegram                              | Prompt for correct department command    | Switch                         |                                    |                                                                                                |
| Google Drive Trigger    | Google Drive Trigger                  | Watch Billing folder for new files       |                                  | Download file                       | ## billing upload                                                                              |
| Download file          | Google Drive                         | Download new Billing files                | Google Drive Trigger            | Pinecone Vector Store3              |                                                                                                |
| Pinecone Vector Store3  | Pinecone Vector Store                 | Insert Billing embeddings                 | Download file                  |                                    |                                                                                                |
| Embeddings Cohere3      | LangChain Embeddings (Cohere)         | Generate embeddings for Billing docs     |                                | Pinecone Vector Store3              |                                                                                                |
| Default Data Loader     | LangChain Document Loader             | Load Billing document content             | Download file                  | Character Text Splitter             |                                                                                                |
| Character Text Splitter | LangChain Text Splitter               | Split Billing document text               | Default Data Loader            | Embeddings Cohere3                 |                                                                                                |
| Google Drive Trigger1   | Google Drive Trigger                  | Watch Tech Support folder for new files  |                                | Download file1                     | ## technical questions                                                                       |
| Download file1         | Google Drive                         | Download new Tech Support files           | Google Drive Trigger1          | Pinecone Vector Store4              |                                                                                                |
| Pinecone Vector Store4  | Pinecone Vector Store                 | Insert Tech Support embeddings            | Download file1                |                                  |                                                                                                |
| Embeddings Cohere4      | LangChain Embeddings (Cohere)         | Generate embeddings for Tech Support docs|                              | Pinecone Vector Store4              |                                                                                                |
| Default Data Loader1    | LangChain Document Loader             | Load Tech Support document content        | Download file1                | Character Text Splitter1            |                                                                                                |
| Character Text Splitter1| LangChain Text Splitter               | Split Tech Support document text          | Default Data Loader1          | Embeddings Cohere4                 |                                                                                                |
| Google Drive Trigger2   | Google Drive Trigger                  | Watch Return Policy folder for new files |                                | Download file2                     | ## return policy                                                                            |
| Download file2         | Google Drive                         | Download new Return Policy files           | Google Drive Trigger2          | Pinecone Vector Store5              |                                                                                                |
| Pinecone Vector Store5  | Pinecone Vector Store                 | Insert Return Policy embeddings            | Download file2                |                                  |                                                                                                |
| Embeddings Cohere5      | LangChain Embeddings (Cohere)         | Generate embeddings for Return Policy docs|                              | Pinecone Vector Store5              |                                                                                                |
| Default Data Loader2    | LangChain Document Loader             | Load Return Policy document content        | Download file2                | Character Text Splitter2            |                                                                                                |
| Character Text Splitter2| LangChain Text Splitter               | Split Return Policy document text          | Default Data Loader2          | Embeddings Cohere5                 |                                                                                                |
| Execute a SQL query     | PostgreSQL Execute Query               | Create user sessions table (initial setup)|                                |                                  | ## just to create table                                                                     |
| Sticky Note             | Sticky Note                          | Annotation for billing upload             |                                |                                  | ## billing upload                                                                           |
| Sticky Note1            | Sticky Note                          | Annotation for technical questions        |                                |                                  | ## technical questions                                                                    |
| Sticky Note2            | Sticky Note                          | Annotation for return policy               |                                |                                  | ## return policy                                                                         |
| Sticky Note3            | Sticky Note                          | Annotation for DB table creation           |                                |                                  | ## just to create table                                                                   |
| Sticky Note4            | Sticky Note                          | Annotation for main bot block              |                                |                                  | ## main bot                                                                              |
| Sticky Note6            | Sticky Note                          | Workflow summary notes                      |                                |                                  | ## working - listens for messages - uses PostgreSQL and Pinecone vector stores - PDFs from Google Drive |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen to "message" updates only  
   - Obtain Telegram Bot API credentials and link  

2. **Set up PostgreSQL Database and Table**  
   - Create table `tg_user_sessions` with columns:  
     - user_id (BIGINT, PRIMARY KEY)  
     - department (TEXT)  
     - active (BOOLEAN, default FALSE)  
     - last_updated (TIMESTAMP, default NOW)  
   - Use Execute a SQL query node with `CREATE TABLE` command  

3. **Insert User Session If Not Exists**  
   - Add PostgreSQL Execute Query node to insert user id with active FALSE if not present  
   - Use Telegram message from.id as user_id parameter  

4. **Select User Session**  
   - Add PostgreSQL Select rows node to fetch user session by user_id  

5. **Add Switch Node (Switch1) to Route Based on Message**  
   - Configure expression output to handle:  
     - "/start" → send welcome message  
     - "/end" → deactivate session  
     - active & department assigned → AI processing  
     - active but no department → route to department command Switch  
     - otherwise prompt to start  

6. **Add Telegram Message Nodes for Welcome and Instructions**  
   - Welcome message node connected to "/start" path  
   - Instruction message node connected to path prompting user to use /start or /end  

7. **Add PostgreSQL Execute Query to Reset Session on /start**  
   - Clear active and department fields for user on /start command  

8. **Add Switch Node to Route Department Commands (/ReturnPolicy, /TechSupport, /billing)**  
   - Connect from Switch1 fallback path for department commands  

9. **Add Telegram Message Nodes to Confirm Department Choice**  
   - For each department command, send confirmation text  

10. **Update Session Department and Active Flag in PostgreSQL**  
    - For each department, add PostgreSQL Execute Query node to set active=TRUE and department=<DeptName>  

11. **Add Telegram Message Node to Prompt Correct Department for Unrecognized Commands**  

12. **Add PostgreSQL Execute Query to Deactivate Session on /end**  

13. **Add Telegram Message Node to Confirm Session Ended and Prompt for /start**  

14. **Add LangChain AI Agent Node**  
    - Input prompt: Telegram message text  
    - Configure system message instructing department respect and sign-off style  
    - Connect to OpenRouter Chat Model node for LLM  
    - Add Simple Memory Buffer node keyed by user_id for session context  
    - Integrate Pinecone Vector Store nodes as tools for each department namespace ("return policy", "tech ques", "billing")  
    - Connect AI Agent output to Telegram Send Message node for replies  

15. **Set up Google Drive Triggers for Each Department Folder**  
    - Configure Google Drive Trigger nodes to watch specific folders monthly for new files  

16. **Add Google Drive Download File Nodes**  
    - Download new files from triggers  

17. **Add LangChain Document Loader Nodes**  
    - Load downloaded file content as documents (binary input)  

18. **Add LangChain Character Text Splitter Nodes**  
    - Split document text into chunks for embedding  

19. **Add LangChain Embeddings Cohere Nodes**  
    - Generate embeddings from text chunks  

20. **Add Pinecone Vector Store Insert Nodes**  
    - Insert embeddings into Pinecone index with department-specific namespace  
    - Configure using Pinecone credentials and index name (e.g., "n8n-proj")  

21. **Add Sticky Notes for Visual Annotations**  
    - Add notes for billing upload, technical questions, return policy, main bot, and DB table creation for clarity  

22. **Set up Credentials**  
    - Telegram Bot API credentials for Telegram nodes  
    - PostgreSQL credentials for database nodes  
    - Google Drive OAuth2 credentials for Drive nodes  
    - Pinecone API key and environment for vector store nodes  
    - Cohere API key for embeddings  
    - OpenRouter API key for chat model  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow integrates Telegram slash commands to direct queries to department-specific knowledge bases using AI agents.     | Main workflow functionality                                                                                     |
| Pinecone vector stores are used as tool integrations for LangChain AI agent to retrieve department documents contextually.      | Vector store integration                                                                                         |
| Google Drive folders are polled monthly to trigger ingestion of new PDFs into Pinecone for up-to-date knowledge bases.          | Document ingestion automation                                                                                     |
| PostgreSQL database maintains user session states (active/inactive) and department assignment for conversation routing.         | Session and state management                                                                                      |
| AI agent system prompt enforces respect for department boundaries to prevent cross-department confusion in answers.             | AI prompt engineering                                                                                            |
| Official n8n documentation for node configuration: https://docs.n8n.io/nodes/                                                    | General reference                                                                                                |
| Pinecone documentation: https://docs.pinecone.io/                                                                               | Vector store setup                                                                                               |
| LangChain node details: https://n8n.io/integrations/n8n-nodes-langchain                                                          | AI agent and embedding nodes                                                                                     |
| Telegram Bot API reference: https://core.telegram.org/bots/api                                                                   | Telegram integration                                                                                             |
| Google Drive API overview: https://developers.google.com/drive/api/v3/about-sdk                                                   | Google Drive triggers and downloads                                                                               |

---

This completes the detailed analysis and reference documentation for the "Multi-Department Support Bot with Slash Commands, Pinecone & Telegram" n8n workflow. The document provides sufficient detail for understanding, modifying, or reproducing the workflow in n8n.