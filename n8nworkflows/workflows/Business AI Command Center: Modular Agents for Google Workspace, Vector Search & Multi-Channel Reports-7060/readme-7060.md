Business AI Command Center: Modular Agents for Google Workspace, Vector Search & Multi-Channel Reports

https://n8nworkflows.xyz/workflows/business-ai-command-center--modular-agents-for-google-workspace--vector-search---multi-channel-reports-7060


# Business AI Command Center: Modular Agents for Google Workspace, Vector Search & Multi-Channel Reports

---

### 1. Workflow Overview

This workflow, titled **"Business AI Command Center: Modular Agents for Google Workspace, Vector Search & Multi-Channel Reports"**, serves as a comprehensive AI-driven automation center designed to interact seamlessly with Google Workspace (primarily Google Sheets and Google Drive), vector databases, and multiple communication channels. It is tailored for business users who want to harness AI to read, modify, analyze, and report on spreadsheet and document data, while integrating advanced AI models for reasoning, memory, and knowledge extraction. The workflow also supports multi-channel notifications via email, Telegram, Slack, and WhatsApp.

The logical workflow divides into the following main blocks:

- **1.1 Google Sheets Data Management:** Nodes to read, write, update, clear, create, and delete sheets or data within Google Sheets, driven by AI commands.
  
- **1.2 AI Interaction and Orchestration:** The core AI agents and models that process inputs, reason, and generate outputs or decisions. This includes several AI language models, agents, and memory nodes.

- **1.3 Google Drive Document Processing & Vector Store:** Nodes for handling files in Google Drive, extracting and splitting content, embedding into vector stores (Supabase), and enabling semantic search.

- **1.4 Multi-Channel Communication & Reporting:** Nodes responsible for generating structured reports, converting Markdown to HTML, and sending messages or notifications through channels like Gmail, Telegram, Slack, and optionally WhatsApp.

- **1.5 Triggers and Workflow Entry Points:** Various triggers to start the workflow manually or via external events on Slack, Gmail, Telegram, WhatsApp, or via webhooks for AI model interaction.

---

### 2. Block-by-Block Analysis

#### 1.1 Google Sheets Data Management

**Overview:**  
This block manages all interactions with Google Sheets, allowing data retrieval, addition, updates, clearing, sheet creation, and sheet deletion via AI-triggered commands. It is central to enabling AI-driven spreadsheet manipulations.

**Nodes Involved:**  
- Google Sheets - Read Data  
- Google Sheets - Add Data  
- Google Sheets - Update Data  
- Google Sheets - Clear Data  
- Google Sheets - Create Sheet  
- Google Sheets - Delete Sheet  
- MCP Server Sheets

**Node Details:**

- **Google Sheets - Read Data**  
  - Type: Google Sheets Tool  
  - Role: Reads data from specified spreadsheet and range.  
  - Configuration: Requires Document_ID, Sheet_Name, and optionally Range.  
  - Input: Triggered by MCP Server Sheets (AI tool).  
  - Output: Data array for processing.  
  - Edge cases: Invalid sheet or range, auth errors, empty data ranges.

- **Google Sheets - Add Data**  
  - Type: Google Sheets Tool  
  - Role: Appends new rows to a sheet.  
  - Configuration: Document_ID, Sheet_Name, Data_To_Add (array/object).  
  - Input: MCP Server Sheets.  
  - Edge cases: Data format mismatch, rate limits.

- **Google Sheets - Update Data**  
  - Type: Google Sheets Tool  
  - Role: Updates specific cells in a sheet.  
  - Configuration: Document_ID, Sheet_Name, Range (e.g., A1:B2), New_Values.  
  - Input: MCP Server Sheets.  
  - Edge cases: Incorrect range, partial updates.

- **Google Sheets - Clear Data**  
  - Type: Google Sheets Tool  
  - Role: Clears data from specified ranges or entire sheets.  
  - Configuration: Document_ID, Sheet_Name, Range (optional).  
  - Input: MCP Server Sheets.  
  - Edge cases: Permanent data loss risk, accidental clears.

- **Google Sheets - Create Sheet**  
  - Type: Google Sheets Tool  
  - Role: Creates new worksheet tabs within existing spreadsheets.  
  - Configuration: Document_ID, New_Sheet_Name, optional Header_Row.  
  - Input: MCP Server Sheets.  
  - Edge cases: Duplicate sheet names, permission errors.

- **Google Sheets - Delete Sheet**  
  - Type: Google Sheets Tool  
  - Role: Deletes entire sheets permanently.  
  - Configuration: Document_ID, Sheet_Name.  
  - Input: MCP Server Sheets.  
  - Edge cases: Irreversible deletion, backup recommended.

- **MCP Server Sheets**  
  - Type: Model Context Protocol (MCP) Trigger  
  - Role: AI entry point for Google Sheets operations; routes natural language commands to appropriate Sheets nodes.  
  - Configuration: Webhook-based, no direct parameters.  
  - Input: External AI requests, user commands.  
  - Output: Calls corresponding Sheets nodes.  
  - Edge cases: Parsing errors, ambiguous commands.

---

#### 1.2 AI Interaction and Orchestration

**Overview:**  
This block holds the core AI processing agents and models, managing reasoning, memory, external knowledge retrieval, and orchestrating multi-step AI workflows.

**Nodes Involved:**  
- MAIN AGENT  
- Knowledge Agent1  
- Reasoning model (recommended)  
- Anthropic Chat Model1  
- OpenAI Chat Model1  
- OpenRouter Chat Model  
- Postgres Chat Memory  
- Postgres Chat Memory1  
- Think1  
- Calculator  
- Reranker Cohere1  
- General knowledge1  
- Linkedin Scraper  
- Message a model in Perplexity1  
- MCP Sheets  
- search about any doc in google drive1

**Node Details:**

- **MAIN AGENT**  
  - Type: Langchain Agent  
  - Role: Primary AI agent handling multi-tool interactions and orchestrating responses.  
  - Inputs: Receives message variable from triggers, AI memory, and other tool outputs.  
  - Outputs: Sends commands to communication nodes for messaging.  
  - Edge cases: Memory overflow, response timeouts.

- **Knowledge Agent1**  
  - Type: Langchain Agent Tool  
  - Role: Specialized agent for structured knowledge access, vector search, and document querying.  
  - Inputs: Vector stores, memory, language models.  
  - Outputs: AI insights for MAIN AGENT.  
  - Edge cases: Data mismatch, vector store latency.

- **Reasoning model (recommended)**  
  - Type: Language Model Chat (OpenRouter)  
  - Role: Provides reasoning and complex NLP capabilities to MAIN AGENT.  
  - Edge cases: Rate limits, API errors.

- **Anthropic Chat Model1 & OpenAI Chat Model1**  
  - Type: Language Model Chat (Anthropic/OpenAI)  
  - Role: Language understanding and generation capabilities for agents.  
  - Edge cases: Quotas, auth errors.

- **OpenRouter Chat Model**  
  - Type: Language Model Chat (OpenRouter)  
  - Role: Used for structured report generation.  
  - Edge cases: API availability.

- **Postgres Chat Memory & Postgres Chat Memory1**  
  - Type: Chat Memory Storage  
  - Role: Persist conversation context and state for agents.  
  - Edge cases: DB connectivity, data corruption.

- **Think1, Calculator**  
  - Type: Tool Nodes  
  - Role: Provide reasoning or calculation capabilities to agents.  
  - Edge cases: Calculation errors.

- **Reranker Cohere1**  
  - Type: AI Reranker  
  - Role: Enhance vector search results by re-ranking relevance.  
  - Edge cases: Latency, API errors.

- **General knowledge1**  
  - Type: Vector Store Supabase  
  - Role: Vector database for general knowledge queries.  
  - Edge cases: Indexing delays.

- **Linkedin Scraper**  
  - Type: Agent Tool  
  - Role: Extract LinkedIn data through AI-powered scraping.  
  - Edge cases: Rate limits, data format changes.

- **Message a model in Perplexity1**  
  - Type: Perplexity AI Tool  
  - Role: Query external AI model Perplexity for additional knowledge.  
  - Edge cases: API downtime.

- **MCP Sheets & search about any doc in google drive1**  
  - Type: MCP Client Tools  
  - Role: Interface for AI agents to access Google Sheets and Drive data respectively.  
  - Edge cases: Permissions, API limits.

---

#### 1.3 Google Drive Document Processing & Vector Store

**Overview:**  
This block processes files from Google Drive, extracts text from PDFs and CSVs, splits text for embeddings, and stores vectors in Supabase for semantic search and AI query support.

**Nodes Involved:**  
- Download file  
- Download File1  
- Search Files from Gdrive  
- Read File From GDrive  
- Default Data Loader1  
- Recursive Character Text Splitter1  
- Embeddings OpenAI1  
- Add to Supabase Vector DB  
- FileType  
- Extract from PDF  
- Extract from CSV  
- Get PDF Response  
- Get CSV Response  
- Analyse Image  
- Transcribe Audio

**Node Details:**

- **Download file & Download File1**  
  - Type: Google Drive File Downloader  
  - Role: Downloads files from Google Drive for processing.  
  - Input: Triggered manually or by operation node.  
  - Edge cases: File not found, access denied.

- **Search Files from Gdrive**  
  - Type: Google Drive Tool  
  - Role: Search and locate files in Google Drive.  
  - Edge cases: Large result sets, permission errors.

- **Read File From GDrive**  
  - Type: Tool Workflow  
  - Role: Reads file content and sends to MCP Server for further AI handling.  
  - Edge cases: File format unsupported.

- **Default Data Loader1**  
  - Type: Document Default Data Loader  
  - Role: Loads document data into AI pipeline for embedding.  
  - Edge cases: File corruption.

- **Recursive Character Text Splitter1**  
  - Type: Text Splitter  
  - Role: Splits large texts into smaller chunks for embeddings.  
  - Edge cases: Splitting errors.

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings  
  - Role: Converts text chunks into vector embeddings.  
  - Edge cases: API limits.

- **Add to Supabase Vector DB**  
  - Type: Vector Store Supabase  
  - Role: Stores embeddings for semantic search.  
  - Edge cases: DB write failures.

- **FileType**  
  - Type: Switch Node  
  - Role: Branches processing based on file type (PDF, CSV, Image, Audio).  
  - Edge cases: Unknown file types.

- **Extract from PDF & Extract from CSV**  
  - Type: File Extractors  
  - Role: Extract text content from PDFs and CSVs respectively.  
  - Edge cases: Extraction errors.

- **Get PDF Response & Get CSV Response**  
  - Type: Set Nodes  
  - Role: Format extracted content for downstream processing.  
  - Edge cases: Empty content.

- **Analyse Image & Transcribe Audio**  
  - Type: OpenAI Tools  
  - Role: Analyze images and transcribe audio files.  
  - Edge cases: Unsupported formats.

---

#### 1.4 Multi-Channel Communication & Reporting

**Overview:**  
This block manages the generation of structured reports and their dispatch through various communication channels including Gmail, Telegram, Slack, and WhatsApp.

**Nodes Involved:**  
- Create reports  
- Create structured Reports  
- Markdown to HTML  
- Send a message (Gmail)  
- Send a message1 (Gmail)  
- Send a message2 (Slack)  
- Send a text message (Telegram)  
- Send message (WhatsApp - disabled)

**Node Details:**

- **Create reports & Create structured Reports**  
  - Type: Tool Workflow & Agent  
  - Role: Generate AI-driven reports from processed data.  
  - Output: Markdown formatted report.

- **Markdown to HTML**  
  - Type: Markdown Node  
  - Role: Converts Markdown reports to HTML for email or web display.  
  - Edge cases: Formatting issues.

- **Send a message (Gmail, Slack, Telegram, WhatsApp)**  
  - Type: Communication Nodes  
  - Role: Send messages via respective platforms.  
  - Configuration: Requires OAuth2 or API credentials.  
  - Edge cases: Auth errors, rate limits, disabled WhatsApp node.

---

#### 1.5 Triggers and Workflow Entry Points

**Overview:**  
This block includes all entry points that initiate the workflow either manually, via external apps, or as webhooks for AI model interaction.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Gmail Trigger  
- Slack Trigger  
- Telegram Trigger  
- WhatsApp Trigger (disabled)  
- MCP Server Sheets (Webhook)  
- Google Drive MCP Server (Webhook)  
- When Executed by Another Workflow (Execute Workflow Trigger)

**Node Details:**

- **Manual Trigger**  
  - Type: Manual Trigger  
  - Role: Starts workflow by user click.  
  - Output: Triggers file download and processing.

- **Gmail, Slack, Telegram, WhatsApp Triggers**  
  - Type: Trigger Nodes for respective platforms  
  - Role: Listen for incoming messages or events to start processing.  
  - Edge cases: Connection drops, auth token expiration.

- **MCP Server Sheets & Google Drive MCP Server**  
  - Type: MCP Webhook Triggers  
  - Role: Receive AI model calls for Google Sheets and Drive operations.  
  - Edge cases: Security and permission misconfigurations.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be invoked as a sub-workflow.  
  - Edge cases: Data passing errors.

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                  |
|-----------------------------|------------------------------------|----------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Google Sheets - Read Data    | Google Sheets Tool                  | Read data from Google Sheets            | MCP Server Sheets              | MCP Server Sheets              | 📊 READ SPREADSHEET DATA... (see 2.1)                                                      |
| Google Sheets - Clear Data   | Google Sheets Tool                  | Clear data ranges or entire sheets      | MCP Server Sheets              | MCP Server Sheets              | 🗑️ CLEAR SPREADSHEET DATA... (see 2.1)                                                     |
| Google Sheets - Add Data     | Google Sheets Tool                  | Append new rows to sheets                | MCP Server Sheets              | MCP Server Sheets              | ➕ ADD NEW DATA... (see 2.1)                                                               |
| Google Sheets - Create Sheet | Google Sheets Tool                  | Create new worksheet tabs                | MCP Server Sheets              | MCP Server Sheets              | 📋 CREATE NEW SHEET... (see 2.1)                                                           |
| Google Sheets - Update Data  | Google Sheets Tool                  | Update specific cells/ranges             | MCP Server Sheets              | MCP Server Sheets              | ✏️ UPDATE EXISTING DATA... (see 2.1)                                                       |
| Google Sheets - Delete Sheet | Google Sheets Tool                  | Delete entire sheets                      | MCP Server Sheets              | MCP Server Sheets              | 🗑️ DELETE SHEET... (see 2.1)                                                               |
| MCP Server Sheets            | MCP Trigger                        | AI entry point for Sheets operations     | External AI Requests           | Google Sheets nodes            | 🚀 MCP TRIGGER... (see 2.1)                                                                |
| MAIN AGENT                  | Langchain Agent                    | Core AI orchestration agent               | Set message variable, AI Memory| Communication nodes            | --                                                                                          |
| Knowledge Agent1             | Langchain Agent Tool               | Specialized AI knowledge retrieval       | Vector Stores, AI Models       | MAIN AGENT                    | --                                                                                          |
| Reasoning model (recommended)| LM Chat OpenRouter                 | Provides reasoning capabilities          | MAIN AGENT                    | MAIN AGENT                    | --                                                                                          |
| Anthropic Chat Model1        | LM Chat Anthropic                  | AI language model                         | Knowledge Agent1              | Knowledge Agent1              | --                                                                                          |
| OpenAI Chat Model1           | LM Chat OpenAI                    | AI language model                         | Linkedin Scraper              | Linkedin Scraper              | --                                                                                          |
| OpenRouter Chat Model        | LM Chat OpenRouter                 | Structured report generation              | Create structured Reports     | Create structured Reports     | --                                                                                          |
| Postgres Chat Memory         | Memory Postgres Chat              | Conversation memory                       | MAIN AGENT                   | MAIN AGENT                   | --                                                                                          |
| Postgres Chat Memory1        | Memory Postgres Chat              | Conversation memory                       | Knowledge Agent1              | Knowledge Agent1              | --                                                                                          |
| Think1                      | Tool Think                       | Reasoning tool                            | MAIN AGENT                   | Knowledge Agent1              | --                                                                                          |
| Calculator                  | Tool Calculator                  | Calculation tool                          | MAIN AGENT                   | Knowledge Agent1              | --                                                                                          |
| Reranker Cohere1            | AI Reranker                     | Re-rank vector search results             | General knowledge1            | General knowledge1            | --                                                                                          |
| General knowledge1           | VectorStore Supabase             | Vector DB for general knowledge           | Embeddings OpenAI2            | Knowledge Agent1              | --                                                                                          |
| Linkedin Scraper            | Agent Tool                      | Scrapes LinkedIn data                      | HTTP Request1                | MAIN AGENT                   | --                                                                                          |
| Message a model in Perplexity1| Perplexity Tool                | External AI knowledge querying             | Knowledge Agent1              | Knowledge Agent1              | --                                                                                          |
| MCP Sheets                  | MCP Client Tool                 | AI client interface for Sheets             | MAIN AGENT                   | MAIN AGENT                   | --                                                                                          |
| search about any doc in google drive1| MCP Client Tool        | AI client interface for Google Drive docs  | Knowledge Agent1              | Knowledge Agent1              | --                                                                                          |
| Download file               | Google Drive File Downloader    | Downloads files from Drive                  | Manual Trigger               | Add to Supabase Vector DB     | --                                                                                          |
| Download File1              | Google Drive File Downloader    | Downloads files from Drive                  | Operation                   | FileType                     | --                                                                                          |
| Search Files from Gdrive    | Google Drive Tool               | Searches files on Drive                      | Google Drive MCP Server       | Google Drive MCP Server       | --                                                                                          |
| Read File From GDrive       | Tool Workflow                  | Reads file content for AI processing         | Google Drive MCP Server       | Google Drive MCP Server       | --                                                                                          |
| Default Data Loader1        | Document Default Data Loader    | Loads documents into AI pipeline             | Recursive Character Text Splitter1 | Add to Supabase Vector DB | --                                                                                          |
| Recursive Character Text Splitter1 | Text Splitter             | Splits text into chunks for embedding        | Default Data Loader1          | Default Data Loader1          | --                                                                                          |
| Embeddings OpenAI1          | OpenAI Embeddings              | Generates vector embeddings                   | Default Data Loader1          | Add to Supabase Vector DB     | --                                                                                          |
| Add to Supabase Vector DB   | Vector Store Supabase          | Stores embeddings for semantic search         | Embeddings OpenAI1, Default Data Loader1 | --                  | --                                                                                          |
| FileType                    | Switch Node                   | Branches by file type for processing           | Download File1               | Extract from PDF, Extract from CSV, Analyse Image, Transcribe Audio | --                                                 |
| Extract from PDF            | File Extractor                | Extracts text from PDF files                    | FileType                    | Get PDF Response             | --                                                                                          |
| Extract from CSV            | File Extractor                | Extracts text from CSV files                    | FileType                    | Get CSV Response             | --                                                                                          |
| Get PDF Response            | Set Node                     | Formats PDF extraction output                   | Extract from PDF             | --                          | --                                                                                          |
| Get CSV Response            | Set Node                     | Formats CSV extraction output                   | Extract from CSV             | --                          | --                                                                                          |
| Analyse Image               | OpenAI Tool                  | Analyzes image content                           | FileType                    | --                          | --                                                                                          |
| Transcribe Audio            | OpenAI Tool                  | Transcribes audio files                           | FileType                    | --                          | --                                                                                          |
| Create reports              | Tool Workflow                | Generates AI-driven reports                       | MAIN AGENT                  | MAIN AGENT                  | --                                                                                          |
| Create structured Reports   | Agent Tool                  | Generates structured AI reports                   | OpenRouter Chat Model       | Markdown to HTML            | --                                                                                          |
| Markdown to HTML            | Markdown Node               | Converts Markdown reports to HTML                 | Create structured Reports   | Send a message               | --                                                                                          |
| Send a message (Gmail)      | Gmail Node                  | Sends email messages                              | Markdown to HTML            | --                          | --                                                                                          |
| Send a message1 (Gmail)     | Gmail Node                  | Sends email messages                              | MAIN AGENT                  | --                          | --                                                                                          |
| Send a message2 (Slack)     | Slack Node                  | Sends Slack messages                              | MAIN AGENT                  | --                          | --                                                                                          |
| Send a text message (Telegram) | Telegram Node             | Sends Telegram messages                           | MAIN AGENT                  | --                          | --                                                                                          |
| Send message (WhatsApp)     | WhatsApp Node (disabled)     | Sends WhatsApp messages                           | MAIN AGENT                  | --                          | Disabled node                                                                             |
| When clicking ‘Execute workflow’ | Manual Trigger          | Manual start trigger                              | --                         | Download file               | --                                                                                          |
| Gmail Trigger               | Gmail Trigger               | Trigger workflow on incoming Gmail messages       | --                         | Set message variable        | --                                                                                          |
| Slack Trigger               | Slack Trigger               | Trigger workflow on Slack events                   | --                         | Set message variable        | --                                                                                          |
| Telegram Trigger            | Telegram Trigger            | Trigger workflow on Telegram messages               | --                         | Set message variable        | --                                                                                          |
| WhatsApp Trigger (disabled) | WhatsApp Trigger (disabled) | Trigger workflow on WhatsApp messages               | --                         | Set message variable        | Disabled node                                                                             |
| MCP Server Sheets           | MCP Trigger                | Webhook for AI Google Sheets interaction            | External AI Requests        | Google Sheets nodes         | --                                                                                          |
| Google Drive MCP Server     | MCP Trigger                | Webhook for AI Google Drive interaction             | External AI Requests        | Google Drive nodes          | --                                                                                          |
| When Executed by Another Workflow | Execute Workflow Trigger | Allows invocation as sub-workflow                    | External workflow calls     | Operation                   | --                                                                                          |
| Set message variable        | Set Node                   | Prepares and sets incoming message for AI processing | Gmail/Slack/Telegram Trigger| MAIN AGENT                  | --                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**  
   - Add **Manual Trigger** node named "When clicking ‘Execute workflow’".  
   - Add **Gmail Trigger**, **Slack Trigger**, **Telegram Trigger** (set credentials accordingly).  
   - Add **WhatsApp Trigger** (disabled by default).  
   - Add **MCP Server Sheets** and **Google Drive MCP Server** nodes as webhook triggers with public webhook URLs.

2. **Google Sheets Data Management Nodes:**  
   - Add six **Google Sheets Tool** nodes named:  
     - "Google Sheets - Read Data"  
     - "Google Sheets - Add Data"  
     - "Google Sheets - Update Data"  
     - "Google Sheets - Clear Data"  
     - "Google Sheets - Create Sheet"  
     - "Google Sheets - Delete Sheet"  
   - Configure each with relevant parameters: Document_ID, Sheet_Name, Range, Data_To_Add, New_Values, or New_Sheet_Name as needed.  
   - Connect all these nodes as AI tools to "MCP Server Sheets".

3. **MCP Server Sheets:**  
   - Add **MCP Trigger** node named "MCP Server Sheets" for Google Sheets AI commands.  
   - Configure webhook credentials and link it as AI tool input/output to Google Sheets nodes.

4. **AI Agents and Models:**  
   - Add **Langchain Agent** node named "MAIN AGENT".  
   - Add supporting agents/tools: "Knowledge Agent1", "Think1", "Calculator", "Reranker Cohere1", "Linkedin Scraper", "Message a model in Perplexity1".  
   - Add language model nodes: "Reasoning model (recommended)" (OpenRouter), "Anthropic Chat Model1", "OpenAI Chat Model1", "OpenRouter Chat Model".  
   - Add **Postgres Chat Memory** nodes to maintain conversation context.  
   - Connect these nodes according to AI tool input/output relationships, ensuring "MAIN AGENT" orchestrates communications.

5. **Google Drive and Document Processing:**  
   - Add **Google Drive** nodes: "Download file", "Download File1", "Search Files from Gdrive".  
   - Add **Tool Workflow** node "Read File From GDrive".  
   - Add **Document Loaders**: "Default Data Loader1", "Recursive Character Text Splitter1".  
   - Add **Embeddings OpenAI1** and **Add to Supabase Vector DB** nodes for embedding and storage.  
   - Add **Switch** node "FileType" branching to "Extract from PDF", "Extract from CSV", "Analyse Image", "Transcribe Audio".  
   - Add **Set** nodes "Get PDF Response" and "Get CSV Response" for formatting.  
   - Connect nodes as per file processing logic.

6. **Multi-Channel Communication:**  
   - Add nodes for sending messages: "Send a message" (Gmail), "Send a message1" (Gmail), "Send a message2" (Slack), "Send a text message" (Telegram), and optionally "Send message" (WhatsApp - disabled).  
   - Add **Markdown to HTML** node for report conversion.  
   - Add **Agent** node "Create structured Reports" and **Tool Workflow** node "Create reports" for report generation.  
   - Connect "MAIN AGENT" to these nodes to dispatch messages.

7. **Message Preparation:**  
   - Add **Set** node "Set message variable" to prepare incoming trigger messages for AI processing.  
   - Connect all triggers to this node, then to "MAIN AGENT".

8. **Sub-Workflow and Workflow Execution:**  
   - Add **Execute Workflow Trigger** node "When Executed by Another Workflow" connected to an "Operation" switch node to handle invocation scenarios.

9. **Credentials Setup:**  
   - Configure OAuth2 credentials for Google Sheets, Google Drive, Gmail, Slack, Telegram, and WhatsApp (if enabled).  
   - Configure API keys for OpenAI, Anthropic, Cohere, Perplexity, and Supabase as required.

10. **Test and Validate:**  
    - Test each trigger independently.  
    - Validate AI commands on Google Sheets and Drive.  
    - Confirm report generation and message dispatch.  
    - Monitor for error handling on API limits, auth errors, and data mismatches.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| 📊 Google Sheets operations include reading, adding, updating, clearing, creating, and deleting data.| See Google Sheets Tool nodes in block 1.1 for AI-driven spreadsheet manipulation.                               |
| 🚀 MCP (Model Context Protocol) triggers enable AI natural language command routing for Sheets/Drive.| MCP Server Sheets and Google Drive MCP Server nodes serve as AI webhook entry points.                           |
| ➕ Vector embeddings use OpenAI embeddings stored in Supabase for semantic search capabilities.       | Embeddings OpenAI1 and Add to Supabase Vector DB nodes handle vector storage and search.                        |
| 📋 Reports generated as Markdown are converted to HTML before sending via email or other channels.    | Create structured Reports agent and Markdown to HTML node enable polished report distribution across platforms. |
| ⚠️ Some nodes (e.g., Google Sheets - Delete Sheet) perform irreversible actions; use with caution.    | Always backup important data before using destructive operations.                                               |
| 🔗 Workflow supports multi-channel notifications: Gmail, Slack, Telegram, WhatsApp (disabled).       | Ensure credentials and webhook URLs are properly configured for each messaging platform.                         |
| 💡 AI language models used include OpenAI, Anthropic, OpenRouter, and external services like Perplexity.| Diverse AI models enable flexible and powerful reasoning and knowledge retrieval.                               |
| 🧠 Conversation memory is maintained via Postgres Chat Memory nodes to allow context-aware AI agents. | Helps maintain continuity in multi-turn AI interactions.                                                        |

---

**Disclaimer:** The provided text is exclusively generated from an n8n automated workflow. It adheres strictly to content policies and contains no illegal or offensive material. All data handled is legal and public.

---