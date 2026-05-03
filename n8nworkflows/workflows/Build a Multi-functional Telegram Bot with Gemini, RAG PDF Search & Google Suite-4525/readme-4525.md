Build a Multi-functional Telegram Bot with Gemini, RAG PDF Search & Google Suite

https://n8nworkflows.xyz/workflows/build-a-multi-functional-telegram-bot-with-gemini--rag-pdf-search---google-suite-4525


# Build a Multi-functional Telegram Bot with Gemini, RAG PDF Search & Google Suite

### 1. Workflow Overview

This workflow implements a multi-functional AI-powered Telegram bot integrating Google Gemini language models, Retrieval-Augmented Generation (RAG) for PDF search, Brave Search for web results, and Google Suite tools (Calendar, Drive, Docs) to provide rich interactive capabilities via Telegram. It supports varied content types such as voice messages, text queries, PDF document searching, calendar event management, and web searches, all orchestrated through a modular architecture.

Logical blocks are grouped as follows:

- **1.1 Input Reception & Preprocessing**  
  Listens for Telegram events (text, voice), handles voice-to-text conversion, and sets up message properties.

- **1.2 Content Type Determination & Routing**  
  Determines the type of user input and routes requests to respective processing blocks.

- **1.3 AI Processing & Response Generation**  
  Utilizes Google Gemini chat models, LangChain agents with tools (Calculator, Think, Date & Time, Brave Search), and Replicate API for voice synthesis to generate responses.

- **1.4 RAG PDF Search & Indexing**  
  Handles PDF search queries by refreshing collection, retrieving documents from Google Drive, performing OCR with Mistral OCR API, embedding content with OpenAI embeddings, indexing with Qdrant vector store, and answering with Gemini for RAG.

- **1.5 Google Suite Integration**  
  Manages Google Calendar events, retrieves holidays, sends birthday lists, and handles Google Drive and Docs operations.

- **1.6 Telegram Response Delivery**  
  Sends generated textual responses, voice messages, search results, PDF search results, and confirmations back via Telegram.

- **1.7 Miscellaneous Helpers**  
  Includes typing indicators, batching and looping over items, and memory buffers for conversational context.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preprocessing

- **Overview:**  
  Captures incoming Telegram events including text and voice messages, downloads voice files, converts audio to text, and prepares content for AI processing.

- **Nodes Involved:**  
  - Listen for incoming events  
  - Send Typing action  
  - Determine content type1  
  - Download voice file  
  - Convert audio to text  
  - Combine content and set properties

- **Node Details:**

  - **Listen for incoming events**  
    - *Type:* telegramTrigger  
    - *Role:* Entry point capturing all Telegram updates/events via webhook.  
    - *Config:* Default trigger with webhook ID; no filters.  
    - *Connections:* Outputs to "Send Typing action" and "Determine content type1".  
    - *Edge Cases:* Telegram webhook failures, network issues.

  - **Send Typing action**  
    - *Type:* telegram  
    - *Role:* Sends "typing" action to Telegram chat to indicate bot is processing.  
    - *Config:* Uses webhook ID; no message content.  
    - *Input:* From "Listen for incoming events".  
    - *Edge Cases:* Telegram API rate limits.

  - **Determine content type1**  
    - *Type:* switch  
    - *Role:* Routes flow based on message content type (text, voice, commands, PDF search, calendar, etc.).  
    - *Config:* Switch rules based on message properties (e.g., message type, commands).  
    - *Input:* From "Listen for incoming events".  
    - *Outputs:* Multiple branches leading to help responder, PDF search, RAG QA chain, Google Calendar, Brave Search, voice download, AI agents, and edit fields.  
    - *Edge Cases:* Unrecognized content types.

  - **Download voice file**  
    - *Type:* telegram  
    - *Role:* Downloads the voice message file from Telegram servers.  
    - *Input:* Triggered when voice message detected.  
    - *Output:* Audio file data to "Convert audio to text".  
    - *Edge Cases:* Telegram file download errors.

  - **Convert audio to text**  
    - *Type:* openAi (LangChain node)  
    - *Role:* Converts audio file to text using OpenAI speech-to-text capabilities.  
    - *Input:* Audio file from voice download.  
    - *Output:* Text transcription to "Combine content and set properties".  
    - *Edge Cases:* Audio format issues, transcription inaccuracies, API errors.

  - **Combine content and set properties**  
    - *Type:* set  
    - *Role:* Aggregates transcribed text and related metadata for downstream AI processing.  
    - *Input:* From "Convert audio to text".  
    - *Output:* Passes to AI Agent for response generation.  
    - *Edge Cases:* Data formatting errors.

---

#### 1.2 Content Type Determination & Routing

- **Overview:**  
  Decides which sub-block or service should handle the user input based on its nature, enabling multi-modal processing.

- **Nodes Involved:**  
  - Determine content type1  
  - Help Responder1  
  - Refresh collection  
  - Get PDF  
  - Question and Answer Chain  
  - Google Calendar 2  
  - Brave Search 2  
  - Download voice file  
  - Edit Fields  
  - AI Agent1

- **Node Details:**

  - **Help Responder1**  
    - *Type:* telegram  
    - *Role:* Sends predefined help or error messages when input is unrecognized or help command is invoked.  
    - *Input:* From "Determine content type1" (help branch).  
    - *Edge Cases:* Telegram API failures.

  - **Refresh collection**  
    - *Type:* httpRequest  
    - *Role:* Refreshes or updates PDF document collection metadata before search.  
    - *Input:* Triggered on relevant commands.  
    - *Output:* Passes to "Search PDFs".  
    - *Edge Cases:* API or endpoint unavailability.

  - **Get PDF**  
    - *Type:* googleDrive  
    - *Role:* Retrieves PDF files from Google Drive based on user query or search.  
    - *Output:* Passes to "Mistral Upload" for OCR.  
    - *Edge Cases:* Google Drive permission issues.

  - **Question and Answer Chain**  
    - *Type:* chainRetrievalQa (LangChain)  
    - *Role:* Performs RAG QA on vectorized PDF content.  
    - *Input:* Query content; uses vector store retriever.  
    - *Output:* Sends answer to "Send RAG Answer via Telegram".  
    - *Edge Cases:* Vector store retrieval failures.

  - **Google Calendar 2**  
    - *Type:* googleCalendar  
    - *Role:* Fetches calendar events or birthday lists.  
    - *Output:* Sends birthday list to "Send Birthday List via Telegram".  
    - *Edge Cases:* OAuth token expiration.

  - **Brave Search 2**  
    - *Type:* braveSearch  
    - *Role:* Performs Brave web search for user queries.  
    - *Output:* Sends results to "Send Brave Search Results via Telegram".  
    - *Edge Cases:* API rate limits.

  - **Edit Fields**  
    - *Type:* set  
    - *Role:* Prepares or modifies fields before Google Drive operations.  
    - *Output:* Chains to Google Drive node.  
    - *Edge Cases:* Data mismatch.

---

#### 1.3 AI Processing & Response Generation

- **Overview:**  
  Executes AI logic using Google Gemini chat models, LangChain agents, and external APIs to generate textual and voice responses.

- **Nodes Involved:**  
  - AI Agent1  
  - AI Agent3  
  - Google Gemini Chat Model  
  - Google Gemini Chat Model1  
  - Think  
  - Think1  
  - Calculator  
  - Calculator1  
  - Date & Time  
  - Date & Time1  
  - Brave Search  
  - Brave Search1  
  - Call Replicate API  
  - Download Replicate Audio  
  - Send AI Voice Response via Telegram  
  - Combine content and set properties (used here as well)  
  - Window Buffer Memory  
  - Window Buffer Memory2  
  - Airbnb MCP / Airbnb MCP1 / Airbnb Tools / Airbnb Tools1 (MCP Client Tools)

- **Node Details:**

  - **AI Agent1 & AI Agent3**  
    - *Type:* LangChain agent  
    - *Role:* Central conversational AI agents managing context, tools, and model calls.  
    - *Input:* User query or transcribed text.  
    - *Output:* Text response or triggers further tool calls (Calculator, Think, Brave Search, etc.).  
    - *Edge Cases:* Model timeout, API quota issues.

  - **Google Gemini Chat Model / Google Gemini Chat Model1**  
    - *Type:* lmChatGoogleGemini (LangChain)  
    - *Role:* Language model for chat completions. Different instances serve different agents.  
    - *Edge Cases:* API connectivity, version compatibility.

  - **Think & Think1**  
    - *Type:* toolThink (LangChain tool)  
    - *Role:* Performs reasoning or multi-step thinking processes.  
    - *Edge Cases:* Tool execution errors.

  - **Calculator & Calculator1**  
    - *Type:* toolCalculator (LangChain tool)  
    - *Role:* Executes arithmetic or calculations requested by user.  
    - *Edge Cases:* Syntax errors in expressions.

  - **Date & Time & Date & Time1**  
    - *Type:* dateTimeTool  
    - *Role:* Provides date/time related information or calculations.  
    - *Edge Cases:* Timezone misconfigurations.

  - **Brave Search & Brave Search1**  
    - *Type:* braveSearch / braveSearchTool  
    - *Role:* Performs web search queries to complement AI responses.  
    - *Edge Cases:* API rate limits, no results.

  - **Call Replicate API**  
    - *Type:* httpRequest  
    - *Role:* Calls Replicate API for voice synthesis from AI text responses.  
    - *Edge Cases:* API failures, latency.

  - **Download Replicate Audio**  
    - *Type:* httpRequest  
    - *Role:* Downloads generated audio from Replicate API.  
    - *Edge Cases:* File not found.

  - **Send AI Voice Response via Telegram**  
    - *Type:* telegram  
    - *Role:* Sends synthesized voice message back to Telegram user.  
    - *Edge Cases:* Telegram file upload issues.

  - **Window Buffer Memory & Window Buffer Memory2**  
    - *Type:* LangChain memoryBufferWindow  
    - *Role:* Maintains conversation history for context-aware responses.  
    - *Edge Cases:* Memory overflow or truncation.

  - **Airbnb MCP / Airbnb Tools variants**  
    - *Type:* mcpClientTool  
    - *Role:* Custom plugins/tools presumably for specific domain tasks (Airbnb-related).  
    - *Edge Cases:* Plugin errors, API failures.

---

#### 1.4 RAG PDF Search & Indexing

- **Overview:**  
  Manages retrieval-augmented generation for PDF documents by indexing PDFs into a vector store and performing OCR as needed.

- **Nodes Involved:**  
  - Refresh collection  
  - Search PDFs  
  - Send PDF Search Results via Telegram  
  - Get PDF  
  - Mistral Upload  
  - Mistral Signed URL  
  - Mistral DOC OCR  
  - Code  
  - Loop Over Items  
  - Embeddings OpenAI / Embeddings OpenAI1  
  - Default Data Loader  
  - Token Splitter  
  - Qdrant Vector Store / Qdrant Vector Store1  
  - Wait  
  - Set page  
  - Vector Store Retriever  
  - Gemini for RAG Answer  
  - Question and Answer Chain  
  - Send RAG Answer via Telegram  
  - Send Qdrant Indexing Confirmation  
  - Sticky Notes related to this block (Sticky Note2, Sticky Note8, etc.)

- **Node Details:**

  - **Refresh collection**  
    - Refreshes metadata or index for PDFs before search.

  - **Search PDFs**  
    - Queries Google Drive for PDFs matching criteria.

  - **Send PDF Search Results via Telegram**  
    - Sends list or results of PDF matches to user.

  - **Get PDF**  
    - Downloads PDF files for processing.

  - **Mistral Upload / Mistral Signed URL / Mistral DOC OCR**  
    - Uploads PDFs to Mistral OCR service, retrieves signed URL, and performs OCR to extract text.

  - **Code**  
    - Custom JavaScript for further processing or formatting OCR data.

  - **Loop Over Items**  
    - Batches items for sequential processing (e.g., multiple PDFs).

  - **Embeddings OpenAI / Embeddings OpenAI1**  
    - Generates vector embeddings of OCRed document text.

  - **Default Data Loader**  
    - Loads documents into LangChain structure for vector store.

  - **Token Splitter**  
    - Splits text into manageable token chunks for embedding.

  - **Qdrant Vector Store / Qdrant Vector Store1**  
    - Vector database for storing and retrieving document embeddings.

  - **Wait**  
    - Wait node to handle async indexing delays.

  - **Set page**  
    - Sets pagination or index page parameters.

  - **Vector Store Retriever**  
    - Retrieves relevant documents from vector store for QA.

  - **Gemini for RAG Answer**  
    - Uses Google Gemini language model to answer queries based on retrieved documents.

  - **Question and Answer Chain**  
    - Orchestrates retrieval and language model for RAG.

  - **Send RAG Answer via Telegram**  
    - Sends the final RAG-based answer back to Telegram.

  - **Send Qdrant Indexing Confirmation**  
    - Notifies user of successful indexing.

  - **Edge Cases:**  
    OCR failures, indexing delays, vector store connection issues, Google Drive access errors.

---

#### 1.5 Google Suite Integration

- **Overview:**  
  Integrates Google Calendar for event creation and holiday retrieval, Google Drive and Docs operations, and sends related data back to Telegram.

- **Nodes Involved:**  
  - Google Calendar Create Event / Google Calendar Create Event1  
  - Google Calendar Next Holiday / Google Calendar Next Holiday1  
  - Google Calendar 2  
  - Send Birthday List via Telegram  
  - Google Drive / Google Drive1 / Google Drive2  
  - Google Docs  
  - Edit Fields  
  - Send Invoice PDF via Telegram

- **Node Details:**

  - **Google Calendar Create Event / Create Event1**  
    - Creates calendar events based on user input.

  - **Google Calendar Next Holiday / Next Holiday1**  
    - Retrieves next official holidays.

  - **Google Calendar 2**  
    - Fetches birthdays/events list.

  - **Send Birthday List via Telegram**  
    - Sends formatted birthday/event list via Telegram.

  - **Google Drive / Google Drive1 / Google Drive2**  
    - Reads and manages files on Google Drive.

  - **Google Docs**  
    - Reads or updates Google Docs files.

  - **Edit Fields**  
    - Prepares data fields before Google Drive operations.

  - **Send Invoice PDF via Telegram**  
    - Sends documents such as invoices back to Telegram users.

  - **Edge Cases:**  
    OAuth token expiration, API rate limits, permission issues.

---

#### 1.6 Telegram Response Delivery

- **Overview:**  
  Ensures all AI-generated text, voice responses, search results, and notifications are delivered back to Telegram users smoothly.

- **Nodes Involved:**  
  - Send AI Text Response (Chat)  
  - Send Brave Search Results via Telegram  
  - Send RAG Answer via Telegram  
  - Send PDF Search Results via Telegram  
  - Send AI Voice Response via Telegram  
  - Send Qdrant Indexing Confirmation  
  - Send Birthday List via Telegram  
  - Send Invoice PDF via Telegram

- **Node Details:**  
  - *Type:* telegram nodes  
  - *Role:* Send respective response types back to the user via Telegram using specific webhook configurations.  
  - *Edge Cases:* Telegram API failures, message size limits.

---

#### 1.7 Miscellaneous Helpers

- **Overview:**  
  Supports memory buffering, batching, typing indicators, and custom tool integrations.

- **Nodes Involved:**  
  - Window Buffer Memory / Window Buffer Memory2  
  - Loop Over Items  
  - Wait  
  - Sticky Notes (various)  
  - Code (custom JS snippets)  
  - Airbnb MCP & Tools (custom domain tools)

- **Node Details:**  
  - Buffer memories maintain conversational context.  
  - Loop Over Items handles batch processing of documents or messages.  
  - Wait node manages timing and rate controls.  
  - Sticky Notes provide inline documentation or reminders.  
  - Airbnb Tools nodes integrate specific MCP client functionalities.

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                            | Input Node(s)                | Output Node(s)                        | Sticky Note                               |
|--------------------------------|---------------------------------|--------------------------------------------|-----------------------------|-------------------------------------|-------------------------------------------|
| Listen for incoming events      | telegramTrigger                 | Entry point for Telegram messages          |                             | Send Typing action, Determine content type1 |                                           |
| Send Typing action              | telegram                       | Sends "typing" indicator                    | Listen for incoming events  |                                     |                                           |
| Determine content type1         | switch                        | Routes requests by content type             | Listen for incoming events  | Multiple branches including Help Responder1, Refresh collection, Get PDF, AI Agent1, etc. |                                           |
| Help Responder1                | telegram                       | Sends help/error messages                    | Determine content type1      |                                     |                                           |
| Refresh collection             | httpRequest                   | Refreshes PDF document collection           | Determine content type1      | Search PDFs                         |                                           |
| Search PDFs                   | googleDrive                   | Searches PDFs in Google Drive                | Refresh collection           | Send PDF Search Results via Telegram |                                           |
| Send PDF Search Results via Telegram | telegram                       | Sends PDF search results to user             | Search PDFs                  |                                     |                                           |
| Get PDF                       | googleDrive                   | Downloads PDF files                           | Determine content type1      | Mistral Upload                     |                                           |
| Mistral Upload                | httpRequest                   | Uploads PDF to OCR service                    | Get PDF                     | Mistral Signed URL                 |                                           |
| Mistral Signed URL            | httpRequest                   | Retrieves signed URL for OCR                  | Mistral Upload              | Mistral DOC OCR                   |                                           |
| Mistral DOC OCR              | httpRequest                   | Performs OCR on uploaded PDF                  | Mistral Signed URL          | Code                              |                                           |
| Code                         | code                          | Processes OCR results                          | Mistral DOC OCR             | Loop Over Items                   |                                           |
| Loop Over Items              | splitInBatches                | Processes items in batches                     | Code                        | Send Qdrant Indexing Confirmation, Set page |                                           |
| Send Qdrant Indexing Confirmation | telegram                       | Confirms successful indexing                   | Loop Over Items             |                                     |                                           |
| Set page                     | set                           | Sets pagination/index parameters               | Loop Over Items             | Qdrant Vector Store               |                                           |
| Qdrant Vector Store          | vectorStoreQdrant             | Stores embeddings into vector DB               | Set page, Embeddings OpenAI | Wait                             |                                           |
| Wait                        | wait                          | Handles async delays                            | Qdrant Vector Store         | Loop Over Items                   |                                           |
| Embeddings OpenAI            | embeddingsOpenAi              | Generates vector embeddings                      | Code                        | Qdrant Vector Store               |                                           |
| Default Data Loader          | documentDefaultDataLoader     | Loads documents for vector store                 | Token Splitter              | Qdrant Vector Store               |                                           |
| Token Splitter              | textSplitterTokenSplitter     | Splits text into tokens for embedding            |                            | Default Data Loader               |                                           |
| Vector Store Retriever       | retrieverVectorStore          | Retrieves relevant docs from vector DB           | Qdrant Vector Store1        | Question and Answer Chain         |                                           |
| Gemini for RAG Answer        | lmChatGoogleGemini            | Answers queries using retrieved docs             | Question and Answer Chain   | Send RAG Answer via Telegram      |                                           |
| Question and Answer Chain    | chainRetrievalQa              | Orchestrates RAG QA process                       | Vector Store Retriever      | Send RAG Answer via Telegram      |                                           |
| Send RAG Answer via Telegram | telegram                       | Sends RAG-based answers to Telegram users          | Question and Answer Chain   |                                 |                                           |
| AI Agent1                   | agent                         | Main AI conversational agent                      | Combine content and set properties, Think, Calculator, Date & Time, Brave Search, etc. | Send AI Text Response (Chat)           |                                           |
| AI Agent3                   | agent                         | AI agent handling voice synthesis and advanced tools | Combine content and set properties, Think1, Calculator1, Date & Time1, Brave Search1, Airbnb Tools, etc. | Call Replicate API                    |                                           |
| Google Gemini Chat Model     | lmChatGoogleGemini            | Provides language model chat completions           | AI Agent1                  | AI Agent1                        |                                           |
| Google Gemini Chat Model1    | lmChatGoogleGemini            | Provides language model chat completions           | AI Agent3                  | AI Agent3                        |                                           |
| Think                      | toolThink                    | Performs reasoning or complex thinking             | AI Agent1                  | AI Agent1                        |                                           |
| Think1                     | toolThink                    | Performs reasoning or complex thinking             | AI Agent3                  | AI Agent3                        |                                           |
| Calculator                 | toolCalculator               | Calculates arithmetic expressions                    | AI Agent1                  | AI Agent1                        |                                           |
| Calculator1                | toolCalculator               | Calculates arithmetic expressions                    | AI Agent3                  | AI Agent3                        |                                           |
| Date & Time                | dateTimeTool                 | Provides date/time info                              | AI Agent1                  | AI Agent1                        |                                           |
| Date & Time1               | dateTimeTool                 | Provides date/time info                              | AI Agent3                  | AI Agent3                        |                                           |
| Brave Search               | braveSearchTool              | Web search via Brave                                | AI Agent1                  | AI Agent1                        |                                           |
| Brave Search1              | braveSearchTool              | Web search via Brave                                | AI Agent3                  | AI Agent3                        |                                           |
| Call Replicate API         | httpRequest                 | Calls Replicate API for voice synthesis              | AI Agent3                  | Download Replicate Audio         |                                           |
| Download Replicate Audio   | httpRequest                 | Downloads synthesized voice audio                     | Call Replicate API          | Send AI Voice Response via Telegram |                                           |
| Send AI Voice Response via Telegram | telegram                       | Sends voice response to Telegram users               | Download Replicate Audio    |                                 |                                           |
| Google Calendar Create Event | googleCalendarTool           | Creates calendar events                               | AI Agent1                  | AI Agent1                        |                                           |
| Google Calendar Next Holiday | googleCalendarTool           | Retrieves next holidays                               | AI Agent1                  | AI Agent1                        |                                           |
| Google Calendar 2           | googleCalendar              | Fetches birthdays/events list                         | Determine content type1      | Send Birthday List via Telegram  |                                           |
| Send Birthday List via Telegram | telegram                       | Sends birthday list to Telegram users                 | Google Calendar 2          |                                 |                                           |
| Google Drive               | googleDrive                 | Manages files on Google Drive                          | Edit Fields                | Google Docs                     |                                           |
| Google Docs                | googleDocs                  | Manages Google Docs files                              | Google Drive               | Google Drive1                   |                                           |
| Google Drive1              | googleDrive                 | Manages files on Google Drive                          | Google Docs                | Google Drive2, Send Invoice PDF via Telegram |                                           |
| Google Drive2              | googleDrive                 | Manages files on Google Drive                          | Google Drive1              |                                 |                                           |
| Send Invoice PDF via Telegram | telegram                       | Sends invoice PDFs to Telegram users                   | Google Drive1              |                                 |                                           |
| Edit Fields                | set                         | Prepares data fields                                   | Determine content type1      | Google Drive                   |                                           |
| Loop Over Items            | splitInBatches              | Batches processing of multiple items                   | Code, Wait                 | Send Qdrant Indexing Confirmation, Set page |                                           |
| Wait                      | wait                        | Handles timing and asynchronous delays                 | Qdrant Vector Store         | Loop Over Items                |                                           |
| Window Buffer Memory       | memoryBufferWindow          | Maintains conversational memory                         |                             | AI Agent1                     |                                           |
| Window Buffer Memory2      | memoryBufferWindow          | Maintains conversational memory                         |                             | AI Agent3                     |                                           |
| Airbnb MCP / Tools variants | mcpClientTool               | Custom tools for domain-specific functionalities        | AI Agent1 / AI Agent3       | AI Agent1 / AI Agent3          |                                           |
| Sticky Notes (various)      | stickyNote                  | Inline documentation and reminders                      |                             |                                 |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger**  
   - Node: "Listen for incoming events" (telegramTrigger)  
   - Configure webhook to receive all Telegram updates.

2. **Send Typing Indicator**  
   - Node: "Send Typing action" (telegram)  
   - Connect from trigger; sends typing action on message receipt.

3. **Add Switch Node for Content Type**  
   - Node: "Determine content type1" (switch)  
   - Configure rules to detect: help commands, PDF search, voice messages, calendar queries, Brave Search, AI agent queries, editing commands.

4. **Help Responder**  
   - Node: "Help Responder1" (telegram)  
   - Sends predefined help text for unrecognized inputs.

5. **PDF Search Flow**  
   - "Refresh collection" (httpRequest) to update PDF metadata.  
   - "Search PDFs" (googleDrive) to find PDFs.  
   - "Send PDF Search Results via Telegram" (telegram).  
   - For detailed retrieval: "Get PDF" (googleDrive) → "Mistral Upload" (httpRequest) → "Mistral Signed URL" (httpRequest) → "Mistral DOC OCR" (httpRequest) → "Code" (code node for processing) → "Loop Over Items" (splitInBatches) → "Embeddings OpenAI" (embeddings) → "Qdrant Vector Store" (vectorstore) → "Wait" (wait node) → "Send Qdrant Indexing Confirmation" (telegram).

6. **RAG QA Chain**  
   - "Vector Store Retriever" (retrieverVectorStore) → "Question and Answer Chain" (chainRetrievalQa) → "Gemini for RAG Answer" (lmChatGoogleGemini) → "Send RAG Answer via Telegram" (telegram).

7. **Voice Message Handling**  
   - "Download voice file" (telegram) → "Convert audio to text" (OpenAI speech-to-text) → "Combine content and set properties" (set) → "AI Agent3" (agent with Google Gemini Chat Model1) → "Call Replicate API" (httpRequest) → "Download Replicate Audio" (httpRequest) → "Send AI Voice Response via Telegram" (telegram).

8. **Text Query AI Agent**  
   - "Combine content and set properties" (set) → "AI Agent1" (agent) configured with:  
     - Tools: Calculator, Think, Date & Time, Brave Search, Google Calendar tools, Airbnb MCP & Tools.  
     - Language Model: "Google Gemini Chat Model" (lmChatGoogleGemini).  
   - Output: "Send AI Text Response (Chat)" (telegram).

9. **Google Suite Integration**  
   - Google Calendar Create Event and Next Holiday nodes connected as tools within AI agents for event creation and holiday queries.  
   - "Google Calendar 2" for birthday/event list retrieval → "Send Birthday List via Telegram".  
   - Google Drive and Docs nodes for file management linked with "Edit Fields" (set) for data preparation.  
   - "Send Invoice PDF via Telegram" sends documents.

10. **Memory Buffer Setup**  
    - Add "Window Buffer Memory" nodes connected to AI Agents to maintain conversation context.

11. **Brave Search Setup**  
    - Add Brave Search nodes as tools accessible by AI agents for web search queries.

12. **Batch Processing and Delays**  
    - Use "Loop Over Items" and "Wait" nodes to handle asynchronous document processing and rate limiting.

13. **Credentials Setup**  
    - Configure Telegram credentials for triggers and sending messages.  
    - Configure OpenAI credentials for speech-to-text and embeddings.  
    - Configure Google OAuth2 credentials for Calendar, Drive, Docs.  
    - Configure Brave Search API credentials.  
    - Configure Replicate API credentials for voice synthesis.  
    - Configure Mistral OCR API keys.

14. **Testing and Validation**  
    - Test each input type: text, voice, PDF query, calendar commands, web search.  
    - Validate error handling for failed API calls or content mismatches.

15. **Add Sticky Notes**  
    - Place sticky notes in the workflow to document sections and nodes as reminders or instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Video guide on building AI-powered Telegram assistants: https://youtu.be/your-video-link (replace with actual link if any)    | General workflow understanding                                                                 |
| Project credit: Workflow inspired by multi-modal AI bot designs leveraging n8n, LangChain, Google Gemini, Replicate, and Brave | Workflow description and architecture overview                                                  |
| Mistral OCR API documentation: https://mistral.ai/docs/ocr                                                                      | Reference for OCR node configuration                                                           |
| Replicate API voice synthesis documentation: https://replicate.com/docs                                                     | For setting up voice generation nodes                                                          |
| Brave Search API details: https://brave.com/search-api/                                                                       | For configuring Brave Search nodes                                                             |
| Google Developer Console for OAuth2 setup: https://console.developers.google.com/                                              | For Google Suite integration credentials                                                       |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.