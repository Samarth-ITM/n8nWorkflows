Document Q&A Chatbot with Gemini AI and Supabase Vector Search for Telegram

https://n8nworkflows.xyz/workflows/document-q-a-chatbot-with-gemini-ai-and-supabase-vector-search-for-telegram-3940


# Document Q&A Chatbot with Gemini AI and Supabase Vector Search for Telegram

### 1. Workflow Overview

This workflow, titled **"Document Q&A Chatbot with Gemini AI and Supabase Vector Search for Telegram"**, implements an advanced AI assistant within Telegram that answers users’ questions based on uploaded PDF documents. It leverages **Google Gemini** for both text understanding and embedding generation, and **Supabase** as a vector database for efficient semantic search of document contents. The workflow handles two main scenarios:

- **Scenario 1: Chatbot Interaction** — Conversational Q&A where users ask questions in Telegram, and the bot responds intelligently using LLM and vector search.
- **Scenario 2: Document Upload and Embedding** — Users upload PDF documents, which are parsed, split, embedded, and stored in Supabase for future retrieval.

Key logical blocks include:

- **1.1 Telegram Input Reception:** Handling incoming Telegram messages and distinguishing between document uploads, text queries, or unsupported inputs.
- **1.2 Document Processing and Embedding:** Downloading PDFs, extracting text, splitting content, generating embeddings, and storing them in Supabase.
- **1.3 Vector Search and AI Question Answering:** Searching the vector store for relevant document chunks and generating formatted answers using Google Gemini.
- **1.4 Message Formatting and Delivery:** Post-processing AI-generated responses for Telegram-compatible HTML formatting, chunking long messages, and delivering them to users.
- **1.5 Optional External Data Integration:** Incorporating real-time data such as weather information upon user request.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception

- **Overview:**  
Receives all incoming messages from users via Telegram, then routes the message based on content type: document uploads, text messages, or unsupported content.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Command Router  
  - Unsupported message

- **Node Details:**

  1. **Telegram Trigger**  
     - *Type:* Telegram Trigger Node  
     - *Role:* Entry point webhook that listens for incoming Telegram messages (specifically "message" updates).  
     - *Configuration:* Listens for all messages, requires Telegram API credentials.  
     - *Input/Output:* Output to Command Router.  
     - *Edge Cases:* Telegram webhook misconfiguration, invalid message formats.

  2. **Command Router**  
     - *Type:* Switch Node  
     - *Role:* Routes messages into three distinct outputs:  
       - `document`: messages containing a document (PDF)  
       - `text`: messages with text content  
       - `extra`: fallback for unsupported messages  
     - *Key Expressions:* Checks for existence of `$json.message.document` or `$json.message.text`.  
     - *Input:* Telegram Trigger output  
     - *Output:* Routes to "Telegram - Download file" for documents, "AI Agent" for text, or "Unsupported message" otherwise.  
     - *Edge Cases:* Missing or malformed message properties; false positives if Telegram sends unexpected message types.

  3. **Unsupported message**  
     - *Type:* Telegram Node  
     - *Role:* Sends a fallback message notifying users their input is unsupported (non-PDF documents or invalid commands).  
     - *Configuration:* Sends fixed text "Unsupported command or file..." to the user's chat ID.  
     - *Input:* Command Router fallback output  
     - *Edge Cases:* Telegram API failures or rate limits.

---

#### 1.2 Document Processing and Embedding

- **Overview:**  
Handles PDF documents sent by users: downloads the file from Telegram, extracts text, splits the text into chunks, generates embeddings with Gemini, and saves these embeddings to Supabase.

- **Nodes Involved:**  
  - Telegram - Download file  
  - Extract from File  
  - Send processing document message  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Embeddings Google Gemini  
  - Supabase - Save Embeddings  
  - Send embedding Started message  
  - Aggregate  
  - Telegram - Embedding Complete

- **Node Details:**

  1. **Telegram - Download file**  
     - *Type:* Telegram Node (file download)  
     - *Role:* Downloads the PDF file from Telegram servers using the file ID from the incoming message.  
     - *Configuration:* Uses `file_id` from Telegram Trigger → Command Router → document path.  
     - *Input:* Document messages from Command Router  
     - *Output:* Passes file binary data to "Extract from File".  
     - *Edge Cases:* File missing or expired on Telegram, download errors.

  2. **Extract from File**  
     - *Type:* Extract From File Node  
     - *Role:* Extracts plain text from the PDF binary file.  
     - *Configuration:* Operation set to "pdf".  
     - *Input:* File binary from Telegram - Download file  
     - *Output:* Extracted text output sent to Supabase - Save Embeddings and Send embedding Started message.  
     - *Edge Cases:* Corrupt PDFs, unsupported PDF features, extraction failures.

  3. **Send processing document message**  
     - *Type:* Telegram Node  
     - *Role:* Sends "Processing document..." message to inform users the workflow is working on the PDF.  
     - *Input:* After Telegram - Download file node (parallel).  
     - *Edge Cases:* Telegram API failures.

  4. **Recursive Character Text Splitter**  
     - *Type:* LangChain Text Splitter Node  
     - *Role:* Splits extracted document text into manageable chunks for embedding and search.  
     - *Configuration:* Default recursive character splitter (splits on paragraphs/sentences).  
     - *Input:* Outputs from Extract from File or Default Data Loader (for embeddings).  
     - *Output:* Chunks sent to Default Data Loader.  
     - *Edge Cases:* Poor splitting if document text is malformed or very short.

  5. **Default Data Loader**  
     - *Type:* LangChain Document Data Loader Node  
     - *Role:* Loads and prepares document chunks into LangChain documents for embedding generation.  
     - *Input:* Chunks from Recursive Character Text Splitter  
     - *Output:* Sent to Embeddings Google Gemini for embedding.  
     - *Edge Cases:* Empty chunks or formatting issues.

  6. **Embeddings Google Gemini**  
     - *Type:* LangChain Embeddings Node  
     - *Role:* Generates vector embeddings of document chunks using Google's Gemini embedding model `models/text-embedding-004`.  
     - *Credentials:* Google Palm API credentials (Gemini) required.  
     - *Input:* Document chunks from Default Data Loader  
     - *Output:* Embeddings passed to Supabase - Save Embeddings and Supabase Vector Store.  
     - *Edge Cases:* API failures, rate limits, credential invalidity.

  7. **Supabase - Save Embeddings**  
     - *Type:* LangChain Vector Store Node (Supabase)  
     - *Role:* Inserts the generated embeddings and associated document text into the Supabase vector table `user_knowledge_base`.  
     - *Credentials:* Supabase API credentials required.  
     - *Input:* Embeddings from Embeddings Google Gemini or Default Data Loader  
     - *Output:* Passes data to Aggregate node.  
     - *Edge Cases:* Database connection failures, data insertion errors.

  8. **Send embedding Started message**  
     - *Type:* Telegram Node  
     - *Role:* Sends a confirmation message with document metadata (number of pages, creator, title, version) after successful extraction.  
     - *Input:* Extract from File output  
     - *Output:* User notification in Telegram.  
     - *Edge Cases:* Missing metadata fields.

  9. **Aggregate**  
     - *Type:* Aggregate Node  
     - *Role:* Acts as a progress flag to indicate embedding save completion; no real aggregation performed.  
     - *Input:* Supabase - Save Embeddings output  
     - *Output:* Triggers Telegram - Embedding Complete node.  
     - *Edge Cases:* None significant.

  10. **Telegram - Embedding Complete**  
      - *Type:* Telegram Node  
      - *Role:* Sends final confirmation message: "✅ Document saved! Feel free to start asking questions about it."  
      - *Input:* Aggregate node  
      - *Edge Cases:* Telegram API failures.

---

#### 1.3 Vector Search and AI Question Answering

- **Overview:**  
Processes user text questions by performing a vector similarity search in Supabase to find relevant document chunks, then uses Google Gemini LLM to generate an informed, formatted answer.

- **Nodes Involved:**  
  - AI Agent  
  - Supabase Vector Store  
  - Answer questions with a vector store  
  - Google Gemini Chat Model  
  - Simple Memory  
  - Think  
  - OpenWeatherMap (optional)

- **Node Details:**

  1. **AI Agent**  
     - *Type:* LangChain Agent Node  
     - *Role:* Central AI node that receives user questions, integrates memory, vector search, and LLM calls to generate answers.  
     - *Configuration:*  
       - Uses Google Gemini Chat Model as LLM.  
       - Integrates vector search tool (Answer questions with a vector store).  
       - Incorporates Simple Memory to maintain session context per user ID.  
       - System prompt instructs to handle Telegram commands, validate input types, use HTML formatting compatible with Telegram, and split messages logically for Telegram limits.  
     - *Input:* Text messages from Command Router (text output)  
     - *Output:* AI-generated output text sent to "Handle formatting and split".  
     - *Edge Cases:* LLM API failures, memory session errors, vector search failures.

  2. **Supabase Vector Store**  
     - *Type:* LangChain Vector Store Node (Supabase)  
     - *Role:* Performs vector similarity search against the `user_knowledge_base` table using user query embeddings.  
     - *Credentials:* Supabase API  
     - *Input:* Query embeddings generated by AI Agent or Google Gemini Chat Model.  
     - *Output:* Relevant document chunks sent to "Answer questions with a vector store".  
     - *Edge Cases:* Query timeouts, empty results.

  3. **Answer questions with a vector store**  
     - *Type:* LangChain Tool Node  
     - *Role:* Uses vector search results to provide context for the AI Agent to generate relevant answers.  
     - *Input:* Vector search results from Supabase Vector Store  
     - *Output:* Feeds context back into AI Agent for response generation.  
     - *Edge Cases:* No matching documents found.

  4. **Google Gemini Chat Model**  
     - *Type:* LangChain LLM Node  
     - *Role:* Provides the underlying large language model capability for AI Agent and vector store tool.  
     - *Model:* `models/gemini-2.5-flash-preview-04-17`  
     - *Credentials:* Google Palm API  
     - *Edge Cases:* API quotas, latency, errors.

  5. **Simple Memory**  
     - *Type:* LangChain Memory Node  
     - *Role:* Maintains conversation history per user (session key based on Telegram user ID) to provide context in AI responses.  
     - *Input:* Receives conversation messages from Telegram Trigger  
     - *Output:* Injects memory context into AI Agent.  
     - *Edge Cases:* Memory overflow, session conflicts.

  6. **Think**  
     - *Type:* LangChain Tool Think Node  
     - *Role:* Internal reasoning step; may be used for invoking intermediate thoughts or managing agent workflow.  
     - *Input:* Connected to AI Agent tool input.  
     - *Edge Cases:* Unusual AI tool errors.

  7. **OpenWeatherMap (optional)**  
     - *Type:* OpenWeatherMap Node  
     - *Role:* Fetches current weather information when requested by the user.  
     - *Credentials:* OpenWeatherMap API  
     - *Input:* Invoked as AI Agent tool when relevant commands or queries detected.  
     - *Edge Cases:* API key invalid, city not found.

---

#### 1.4 Message Formatting and Delivery

- **Overview:**  
Post-processes AI-generated HTML responses to ensure compatibility with Telegram’s supported HTML markup, escapes special characters, splits long responses into multiple messages respecting Telegram’s 4096-character limit, and sends responses back to users.

- **Nodes Involved:**  
  - Handle formatting and split (Code node)  
  - Split Out  
  - Manual Mapping  
  - Telegram  
  - Fallback - No formatting

- **Node Details:**

  1. **Handle formatting and split**  
     - *Type:* Code Node (Python)  
     - *Role:* Cleans AI-generated HTML by removing unsupported tags (like `<p>`, `<div>`, `<span>` without specific classes, etc.), escaping special characters in text content, and splitting the cleaned text into message chunks that comply with Telegram's 4096-character limit.  
     - *Key Logic:*  
       - Uses regex to remove unsupported HTML tags while preserving necessary ones.  
       - Escapes `<`, `>`, and `&` characters in text nodes only.  
       - Splits messages on logical boundaries (e.g., paragraphs, blockquotes, code blocks) to avoid breaking tags mid-message.  
     - *Input:* AI Agent output JSON with field `output` (the raw AI text).  
     - *Output:* List of cleaned and chunked messages under `output` key.  
     - *Edge Cases:* Complex or malformed HTML may break regex; splitting may not preserve perfect HTML integrity; fallback node addresses failures.

  2. **Split Out**  
     - *Type:* Split Out Node  
     - *Role:* Splits the array of message chunks into individual items for sequential sending.  
     - *Input:* Output array from "Handle formatting and split".  
     - *Output:* Sends individual messages to Manual Mapping node.  
     - *Edge Cases:* Empty message arrays.

  3. **Manual Mapping**  
     - *Type:* Set Node  
     - *Role:* Maps message parts to `text` and sets the target `chatId` for Telegram message sending.  
     - *Input:* Split messages from Split Out  
     - *Output:* Prepares data for Telegram node.  
     - *Edge Cases:* Missing chat ID.

  4. **Telegram**  
     - *Type:* Telegram Node  
     - *Role:* Sends each formatted message chunk to the user's Telegram chat with HTML parse mode enabled (ensures rich formatting).  
     - *Input:* Manual Mapping output  
     - *Error Handling:* On error, continues to "Fallback - No formatting".  
     - *Edge Cases:* Telegram API limits, message too long, invalid HTML tags.

  5. **Fallback - No formatting**  
     - *Type:* Telegram Node  
     - *Role:* Sends the AI response as plain text without HTML formatting if the formatted message fails to deliver.  
     - *Input:* Error branch from Telegram node.  
     - *Edge Cases:* Telegram failures or message truncation.

---

#### 1.5 Optional External Data Integration (Weather)

- **Overview:**  
Supports the AI agent’s ability to fetch real-time weather data from OpenWeatherMap when requested by the user.

- **Nodes Involved:**  
  - OpenWeatherMap  
  - AI Agent (tool integration)

- **Node Details:**

  1. **OpenWeatherMap**  
     - *Type:* OpenWeatherMap Node  
     - *Role:* Queries current weather for a city provided by the AI Agent via dynamic expression.  
     - *Input:* City name extracted from AI Agent's input or user command.  
     - *Output:* Weather data returned to AI Agent for response generation.  
     - *Edge Cases:* Invalid city names, API key issues.

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                              | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                           |
|----------------------------|--------------------------------------------|----------------------------------------------|-----------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------|
| Telegram Trigger           | Telegram Trigger                           | Entry point for incoming Telegram messages   | —                           | Command Router                  |                                                                                                     |
| Command Router             | Switch                                     | Routes messages by type: document, text, other | Telegram Trigger            | Telegram - Download file, AI Agent, Unsupported message |                                                                                                     |
| Unsupported message        | Telegram                                   | Sends fallback for unsupported inputs        | Command Router               | —                              |                                                                                                     |
| Telegram - Download file   | Telegram                                   | Downloads uploaded PDF file from Telegram     | Command Router (document)    | Extract from File, Send processing document message |                                                                                                     |
| Extract from File          | Extract From File                          | Extracts text from downloaded PDF             | Telegram - Download file     | Supabase - Save Embeddings, Send embedding Started message |                                                                                                     |
| Send processing document message | Telegram                           | Notifies user that PDF processing started     | Telegram - Download file     | —                              |                                                                                                     |
| Recursive Character Text Splitter | LangChain Text Splitter           | Splits extracted text into chunks              | Default Data Loader          | Default Data Loader             |                                                                                                     |
| Default Data Loader        | LangChain Document Data Loader             | Prepares document chunks for embedding        | Recursive Character Text Splitter | Embeddings Google Gemini       |                                                                                                     |
| Embeddings Google Gemini   | LangChain Embeddings                       | Generates embeddings from document chunks     | Default Data Loader          | Supabase - Save Embeddings, Supabase Vector Store |                                                                                                     |
| Supabase - Save Embeddings | LangChain Vector Store                     | Saves embeddings and text chunks in Supabase | Embeddings Google Gemini, Extract from File | Aggregate                      |                                                                                                     |
| Send embedding Started message | Telegram                              | Sends document metadata after processing      | Extract from File            | —                              |                                                                                                     |
| Aggregate                 | Aggregate                                  | Flags completion of embedding save            | Supabase - Save Embeddings   | Telegram - Embedding Complete   |                                                                                                     |
| Telegram - Embedding Complete | Telegram                              | Confirms document saved, ready for Q&A        | Aggregate                   | —                              | Sticky Note10 (Scenario 2 – Document Upload and Embedding flow)                                    |
| AI Agent                  | LangChain Agent                            | Main AI processing: answers questions          | Command Router (text), Think, Answer questions with a vector store, Google Gemini Chat Model, Simple Memory, OpenWeatherMap | Handle formatting and split     | Sticky Note11 (Scenario 1 – Chatbot Interaction)                                                   |
| Supabase Vector Store     | LangChain Vector Store                     | Performs vector similarity search in Supabase | Embeddings Google Gemini     | Answer questions with a vector store |                                                                                                     |
| Answer questions with a vector store | LangChain Tool                   | Provides document context for AI Agent         | Supabase Vector Store        | AI Agent                       |                                                                                                     |
| Google Gemini Chat Model  | LangChain LLM                              | LLM for AI Agent and vector store tool         | —                           | AI Agent, Answer questions with a vector store |                                                                                                     |
| Simple Memory             | LangChain Memory Buffer Window             | Maintains user session memory                   | Telegram Trigger             | AI Agent                      |                                                                                                     |
| Think                     | LangChain Tool Think                       | Internal reasoning tool for AI Agent           | —                           | AI Agent                      |                                                                                                     |
| OpenWeatherMap            | OpenWeatherMap Node                        | Fetches weather data on user request           | AI Agent                    | AI Agent                      |                                                                                                     |
| Handle formatting and split | Code (Python)                            | Cleans and splits AI response for Telegram HTML | AI Agent                    | Split Out                     |                                                                                                     |
| Split Out                 | Split Out                                  | Splits array of message parts into single messages | Handle formatting and split | Manual Mapping                |                                                                                                     |
| Manual Mapping            | Set Node                                   | Sets text and chatId for Telegram message send | Split Out                   | Telegram                      |                                                                                                     |
| Telegram                  | Telegram                                   | Sends formatted messages to user                | Manual Mapping              | Fallback - No formatting (on error) |                                                                                                     |
| Fallback - No formatting  | Telegram                                   | Sends plain text message if formatted send fails | Telegram                    | —                              |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for `"message"` updates only.  
   - Connect your Telegram API credentials.

2. **Add Command Router (Switch Node)**  
   - Add two outputs plus fallback:  
     - Output "document" if `$json.message.document` exists  
     - Output "text" if `$json.message.text` exists  
     - Fallback output for others  
   - Connect Telegram Trigger output to this node.

3. **Add Unsupported message Telegram Node**  
   - Sends fixed text: "Unsupported command or file. 😓 Please upload a valid PDF document or ask your question regarding your files."  
   - Connect Command Router fallback output here.  
   - Use Telegram credentials.

4. **Add Telegram - Download file Node**  
   - Type: Telegram  
   - Set resource: "file"  
   - File ID: `{{$json.message.document.file_id}}`  
   - Connect Command Router "document" output here.

5. **Add Send processing document message Telegram Node**  
   - Text: `<b>Processing document...</b>\n<b>Please wait...⏳</b>`  
   - Parse mode: HTML  
   - Connect parallel output from Telegram - Download file node.

6. **Add Extract from File Node**  
   - Operation: PDF  
   - Input: binary data from Telegram - Download file  
   - Connect Telegram - Download file main output to this node.

7. **Add Send embedding Started message Telegram Node**  
   - Text with document metadata placeholders (e.g. `Num of pages: {{ $json.numpages }}`)  
   - Parse mode: HTML  
   - Connect Extract from File output to this node.

8. **Add Recursive Character Text Splitter Node**  
   - Use default recursive character splitting settings.  
   - Connect to Default Data Loader.

9. **Add Default Data Loader Node**  
   - Use default settings.  
   - Connect Recursive Character Text Splitter output here.

10. **Add Embeddings Google Gemini Node**  
    - Model: `models/text-embedding-004`  
    - Connect Default Data Loader output here.  
    - Use Google Palm API credentials.

11. **Add Supabase - Save Embeddings Node**  
    - Table: `user_knowledge_base`  
    - Mode: Insert  
    - Connect Embeddings Google Gemini and Extract from File outputs here.  
    - Use Supabase API credentials.

12. **Add Aggregate Node**  
    - Default settings, no aggregation fields required.  
    - Connect Supabase - Save Embeddings output here.

13. **Add Telegram - Embedding Complete Node**  
    - Text: `✅ Document saved! Feel free to start asking questions about it.`  
    - Connect Aggregate output here.

14. **Add AI Agent Node**  
    - Text input: `{{$json.message.text}}` (from Command Router "text" output)  
    - Configure with:  
      - System message instructing to handle Telegram commands, format output with Telegram-compatible HTML, escape special characters, and split messages logically.  
      - Prompt type: Define with output parser enabled.  
    - Connect Command Router "text" output here.  
    - Integrate tools: Google Gemini Chat Model, Supabase Vector Store, Answer questions with a vector store, Simple Memory, Think, OpenWeatherMap (optional).  
    - Use Google Palm API credentials.

15. **Add Google Gemini Chat Model Node**  
    - Model: `models/gemini-2.5-flash-preview-04-17`  
    - Use Google Palm API credentials.  
    - Connect AI Agent and Answer questions with a vector store inputs here.

16. **Add Supabase Vector Store Node**  
    - Table: `user_knowledge_base`  
    - Query function: `match_documents`  
    - Use Supabase API credentials.  
    - Connect Embeddings Google Gemini output and AI Agent tool inputs.

17. **Add Answer questions with a vector store Node**  
    - Description: Use as context if question relates to uploaded documents.  
    - Connect Supabase Vector Store output here.  
    - Connect output back to AI Agent.

18. **Add Simple Memory Node**  
    - Session Key: `{{$json.message.from.id}}` (Telegram user ID)  
    - Connect Telegram Trigger output here.  
    - Connect output to AI Agent memory input.

19. **Add Think Node**  
    - Connect to AI Agent tool input for internal reasoning.

20. **Add OpenWeatherMap Node** (optional)  
    - City name sourced dynamically from AI Agent input.  
    - Use OpenWeatherMap API credentials.  
    - Connect AI Agent tool input and output.

21. **Add Handle formatting and split Code Node**  
    - Write Python code to:  
      - Remove unsupported HTML tags based on Telegram documentation  
      - Escape special characters in text content only  
      - Split cleaned text into <=4096 character chunks on logical boundaries  
    - Input: AI Agent output  
    - Output: Array of message chunks.

22. **Add Split Out Node**  
    - Field to split out: `output` (message chunks array)  
    - Connect Handle formatting and split output.

23. **Add Manual Mapping (Set) Node**  
    - Map `text` to current split message chunk.  
    - Set `chatId` to Telegram user chat ID from Command Router message.  
    - Connect Split Out output.

24. **Add Telegram Node**  
    - Send messages with HTML parse mode enabled.  
    - Connect Manual Mapping output.  
    - On error, continue to fallback node.

25. **Add Fallback - No formatting Telegram Node**  
    - Sends AI response as plain text without HTML, fallback for Telegram send errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| ✅ Scenario 2 – Document Upload and Embedding: Flow for downloading a document sent via Telegram, extracting its text, generating embeddings, and inserting them into Supabase Vector Store.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note10 in workflow                                                                                           |
| ✅ Scenario 1 – Chatbot Interaction: Flow for handling user messages sent to the bot. Includes accessing weather data, answering questions based on user-uploaded documents, and running code using a code execution tool.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note11 in workflow                                                                                           |
| This project transforms a Telegram bot into an AI assistant for your documents, leveraging Google Gemini, Supabase vector search, and n8n automation with no code. Upload PDFs, ask questions, and receive rich HTML-formatted answers split into Telegram message-size chunks. Optional weather integration included.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note (large detailed project description node)                                                              |
| Setup includes creating Telegram, Google Gemini, and Supabase API credentials, importing this workflow into your n8n instance, and preparing your Supabase vector table with the provided SQL commands. The workflow is designed for personal, single-user scenarios and does not manage multi-user sessions or persistent state beyond simple memory buffer per user.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Setup section in Sticky Note                                                                                         |
| Video demo is available: "Unleashing AI on My Bookshelf: Flow Programming Powers a Next-Level Telegram Bot" on YouTube, accessible by clicking the bot image in the sticky note.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | https://www.youtube.com/watch?v=r_KGyJApy5M                                                                         |
| Supabase vector search uses pgvector extension with a custom SQL function `match_documents` that performs cosine similarity searches to rank document chunks by relevance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Embedded SQL script in Sticky Note                                                                                   |
| Telegram supports limited HTML tags; unsupported tags are removed from LLM output by Python code to avoid broken messages. Ensure all `<`, `>`, and `&` characters in text (not tags) are escaped.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Explanation in system prompt and Handle formatting and split node comments                                           |
| The workflow uses a session-based memory buffer keyed by Telegram user ID to maintain conversation context per user, but does not support multi-user session isolation or concurrency.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Simple Memory node description                                                                                       |
| For multi-user or production-grade Telegram bots with session management, see: https://github.com/mohamadghaffari/gemini-tel-bot                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Mentioned in Sticky Note                                                                                              |

---

This documentation provides a detailed understanding of the workflow structure, individual node purposes, and a comprehensive guide to recreate or modify the chatbot system. It anticipates integration challenges such as API limits, unsupported HTML formatting, and document parsing issues, ensuring robust deployment and maintenance.