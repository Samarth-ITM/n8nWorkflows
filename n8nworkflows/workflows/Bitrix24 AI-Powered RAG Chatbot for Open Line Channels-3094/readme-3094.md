Bitrix24 AI-Powered RAG Chatbot for Open Line Channels

https://n8nworkflows.xyz/workflows/bitrix24-ai-powered-rag-chatbot-for-open-line-channels-3094


# Bitrix24 AI-Powered RAG Chatbot for Open Line Channels

---

### 1. Workflow Overview

This workflow implements an AI-powered Retrieval-Augmented Generation (RAG) chatbot integrated with Bitrix24 Open Line channels. It enables automated, document-based customer support by processing uploaded documents, indexing their content semantically, and responding to user queries with AI-generated answers grounded in the indexed knowledge base.

**Target Use Cases:**
- Customer service teams handling repetitive inquiries
- Support departments with extensive documentation
- Sales teams needing quick product information access
- Organizations aiming for 24/7 automated support in Bitrix24

**Logical Blocks:**

- **1.1 Webhook Handler & Authentication:** Receives incoming Bitrix24 Open Line events, validates tokens, and routes events.
- **1.2 Event Routing & Processing:** Routes events like new messages, bot joins, app installs, and bot deletions to dedicated handlers.
- **1.3 Bot Registration:** Registers the chatbot with Bitrix24 upon installation.
- **1.4 Message Processing & AI Response Generation:** Processes user messages, retrieves relevant document chunks via semantic search, and generates AI responses using Google Gemini.
- **1.5 Document Management & Vector Indexing:** Retrieves documents from Bitrix24 storage, downloads, processes (chunking and embedding), and stores embeddings in Qdrant vector database.
- **1.6 Response Delivery:** Sends generated responses back to Bitrix24 Open Line channels.
- **1.7 Subworkflow Execution & Parameter Merging:** Supports modular execution and parameter passing for complex operations.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Handler & Authentication

- **Overview:**  
  This block receives HTTP POST requests from Bitrix24 Open Line channels, extracts authentication credentials, and validates the application token to ensure request legitimacy.

- **Nodes Involved:**  
  - Bitrix24 Handler (Webhook)  
  - Credentials (Set)  
  - Validate Token (If)  
  - Error Response (Respond to Webhook)

- **Node Details:**

  - **Bitrix24 Handler**  
    - Type: Webhook  
    - Role: Entry point for incoming Bitrix24 events via POST at path `bitrix24/openchannel-rag-bothandler.php`.  
    - Configuration: HTTP POST, response mode set to respond via a node.  
    - Inputs: External HTTP requests.  
    - Outputs: Passes request data to Credentials node.  
    - Edge Cases: Invalid HTTP methods, malformed requests.

  - **Credentials**  
    - Type: Set  
    - Role: Extracts and sets authentication tokens and configuration parameters from the incoming request JSON.  
    - Configuration: Assigns CLIENT_ID, CLIENT_SECRET, application_token, domain, access_token, storageName, folderName from request body or static values.  
    - Inputs: Bitrix24 Handler output.  
    - Outputs: Passes enriched JSON to Validate Token.  
    - Edge Cases: Missing or malformed auth fields.

  - **Validate Token**  
    - Type: If  
    - Role: Validates that the CLIENT_ID matches the application_token or allows bypass (always true condition).  
    - Configuration: Checks equality between CLIENT_ID and application_token or a fallback condition.  
    - Inputs: Credentials output.  
    - Outputs: Routes to Route Event node if valid, else to Error Response.  
    - Edge Cases: Token mismatch leads to 401 error.

  - **Error Response**  
    - Type: Respond to Webhook  
    - Role: Returns HTTP 401 with JSON error message for invalid tokens.  
    - Configuration: Response code 401, JSON body with error details.  
    - Inputs: Validate Token failure branch.  
    - Outputs: Ends workflow execution for unauthorized requests.

---

#### 2.2 Event Routing & Processing

- **Overview:**  
  Routes incoming Bitrix24 events based on their type to dedicated processing nodes for message handling, bot joining, installation, or deletion.

- **Nodes Involved:**  
  - Route Event (Switch)  
  - Process Message (Function)  
  - Process Join (Function)  
  - Process Install (Function)  
  - Process Delete (NoOp)  
  - Send Join Message (HTTP Request)  
  - Register Bot (HTTP Request)  
  - Success Response (Respond to Webhook)

- **Node Details:**

  - **Route Event**  
    - Type: Switch  
    - Role: Routes based on `body.event` field to one of four outputs: ONIMBOTMESSAGEADD, ONIMBOTJOINCHAT, ONAPPINSTALL, ONIMBOTDELETE.  
    - Configuration: Exact string matching for event types.  
    - Inputs: Validate Token success output.  
    - Outputs: Routes to respective processing nodes.  
    - Edge Cases: Unknown event types are not handled explicitly.

  - **Process Message**  
    - Type: Function  
    - Role: Extracts message text, dialog ID, session ID, bot ID, user ID, and auth tokens from event data.  
    - Configuration: Custom JavaScript to parse nested JSON fields.  
    - Inputs: Route Event ONIMBOTMESSAGEADD output.  
    - Outputs: Prepares structured JSON for AI processing.  
    - Edge Cases: Missing fields or unexpected JSON structure.

  - **Process Join**  
    - Type: Function  
    - Role: Prepares a welcome menu message when the bot joins a chat.  
    - Configuration: Returns a fixed menu string with options for users.  
    - Inputs: Route Event ONIMBOTJOINCHAT output.  
    - Outputs: JSON with dialog ID, message, and auth info.  
    - Edge Cases: None significant.

  - **Process Install**  
    - Type: Function  
    - Role: Prepares bot registration parameters upon app installation.  
    - Configuration: Sets bot code, type, events URLs, and bot properties like name, color, email.  
    - Inputs: Route Event ONAPPINSTALL output.  
    - Outputs: JSON with registration data and auth tokens.  
    - Edge Cases: Missing webhook URL or auth data.

  - **Process Delete**  
    - Type: NoOp  
    - Role: Placeholder for bot deletion event handling.  
    - Inputs: Route Event ONIMBOTDELETE output.  
    - Outputs: Passes to Success Response.  
    - Edge Cases: None.

  - **Send Join Message**  
    - Type: HTTP Request  
    - Role: Sends the welcome menu message to Bitrix24 chat via API.  
    - Configuration: POST to `imbot.message.add` endpoint with dialog ID, message, and auth token.  
    - Inputs: Process Join output.  
    - Outputs: Success Response.  
    - Edge Cases: API errors, invalid tokens.

  - **Register Bot**  
    - Type: HTTP Request  
    - Role: Registers the bot with Bitrix24 using parameters from Process Install.  
    - Configuration: POST to `imbot.register` endpoint with bot properties and event URLs.  
    - Inputs: Process Install output.  
    - Outputs: Success Response and merges parameters for subworkflow.  
    - Edge Cases: API failures, duplicate registration.

  - **Success Response**  
    - Type: Respond to Webhook  
    - Role: Returns HTTP 200 with JSON `{ "result": true }` to acknowledge successful processing.  
    - Inputs: From Send Join Message, Register Bot, Send Message, Process Delete, Execute Subworkflow.  
    - Outputs: Ends workflow execution.

---

#### 2.3 Message Processing & AI Response Generation

- **Overview:**  
  Processes user messages by retrieving relevant document chunks from the vector store and generating AI responses using Google Gemini, maintaining conversation context.

- **Nodes Involved:**  
  - Question and Answer Chain (LangChain Retrieval QA)  
  - Vector Store Retriever (LangChain Retriever)  
  - Google Gemini Chat Model (LangChain LLM)  
  - Prepare output parameters (Set)  
  - Send Message (HTTP Request)

- **Node Details:**

  - **Question and Answer Chain**  
    - Type: LangChain Retrieval QA Chain  
    - Role: Uses the user message as a query, retrieves relevant document context, and generates an AI answer.  
    - Configuration:  
      - Input text from `MESSAGE_ORI` field.  
      - System prompt instructs to answer only with specified key-value pairs (`DIALOG_ID`, `AUTH`, `DOMAIN`, `MESSAGE`).  
      - Special response for "find out more about me" query.  
    - Inputs: Process Message output.  
    - Outputs: AI-generated response JSON.  
    - Edge Cases: No relevant documents found, LLM timeouts, malformed responses.

  - **Vector Store Retriever**  
    - Type: LangChain Vector Store Retriever  
    - Role: Retrieves top 10 relevant document chunks from Qdrant vector store based on the query.  
    - Configuration: `topK` set to 10, uses Qdrant collection "bitrix-docs".  
    - Inputs: Connected to Qdrant Vector Store1 output.  
    - Outputs: Provides context to QA Chain.  
    - Edge Cases: Empty or unavailable vector store, API errors.

  - **Google Gemini Chat Model**  
    - Type: LangChain LLM Chat Model  
    - Role: Generates natural language responses based on retrieved context.  
    - Configuration: Uses model `models/gemini-2.0-flash`.  
    - Inputs: Question and Answer Chain.  
    - Outputs: AI response text.  
    - Edge Cases: API rate limits, authentication errors.

  - **Prepare output parameters**  
    - Type: Set  
    - Role: Cleans AI response text by removing markdown and trailing backticks, wraps it in a `data` object for sending.  
    - Inputs: Question and Answer Chain output.  
    - Outputs: Prepares data for Send Message node.  
    - Edge Cases: Unexpected text formatting.

  - **Send Message**  
    - Type: HTTP Request  
    - Role: Sends the AI-generated message back to Bitrix24 chat via `imbot.message.add` API.  
    - Configuration: POST with dialog ID, message text, and auth token.  
    - Inputs: Prepare output parameters output.  
    - Outputs: Success Response.  
    - Edge Cases: API failures, invalid dialog IDs.

---

#### 2.4 Document Management & Vector Indexing

- **Overview:**  
  Retrieves documents from Bitrix24 storage, downloads and processes PDFs by splitting into chunks, generates embeddings via Ollama API, and stores vectors in Qdrant for semantic search.

- **Nodes Involved:**  
  - Execute Workflow Trigger (Trigger)  
  - Get a list of available storages (HTTP Request)  
  - Get a list of List of Files and Folders (HTTP Request)  
  - Get a list of Folders files (HTTP Request)  
  - Split Out folder files and folders (SplitOut)  
  - Filter for files (Filter)  
  - Download file (HTTP Request)  
  - Move files to Vector stored folder (HTTP Request)  
  - Default Data Loader (LangChain Document Loader)  
  - Recursive Character Text Splitter (LangChain Text Splitter)  
  - Embeddings Ollama (LangChain Embeddings)  
  - Qdrant Vector Store (LangChain Vector Store)  
  - Sticky Note (Documentation)

- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Initiates document processing workflow.  
    - Inputs: Manual or scheduled trigger.  
    - Outputs: Starts storage retrieval.

  - **Get a list of available storages**  
    - Type: HTTP Request  
    - Role: Retrieves Bitrix24 storage entities filtered by name (e.g., "Shared drive").  
    - Configuration: POST with filter on ENTITY_TYPE and storageName.  
    - Inputs: Execute Workflow Trigger output.  
    - Outputs: Storage list for next step.  
    - Edge Cases: API errors, empty storage list.

  - **Get a list of List of Files and Folders**  
    - Type: HTTP Request  
    - Role: Retrieves child folders filtered by folderName (e.g., "Open line chat bot documents").  
    - Configuration: POST with parent storage ID and folder name filter.  
    - Inputs: Storage list output.  
    - Outputs: Folder list for next step.  
    - Edge Cases: Folder not found.

  - **Get a list of Folders files**  
    - Type: HTTP Request  
    - Role: Retrieves files inside the identified folder.  
    - Configuration: POST with folder ID.  
    - Inputs: Folder list output.  
    - Outputs: List of files and folders.  
    - Edge Cases: Empty folder, API errors.

  - **Split Out folder files and folders**  
    - Type: SplitOut  
    - Role: Splits the list of files and folders into individual items for processing.  
    - Inputs: Folder files list.  
    - Outputs: Individual file/folder items.  
    - Edge Cases: Empty input.

  - **Filter for files**  
    - Type: Filter  
    - Role: Filters only items of type "file" for further processing.  
    - Inputs: Split Out output.  
    - Outputs: Files only.  
    - Edge Cases: No files found.

  - **Download file**  
    - Type: HTTP Request  
    - Role: Downloads the file content using the provided download URL.  
    - Inputs: Filter for files output.  
    - Outputs: Binary file data.  
    - Edge Cases: Download failures, invalid URLs.

  - **Move files to Vector stored folder**  
    - Type: HTTP Request  
    - Role: Moves processed files to a designated subfolder for vector storage.  
    - Configuration: POST with file ID and target folder ID.  
    - Inputs: Download file output.  
    - Outputs: Confirmation of move.  
    - Edge Cases: Permission errors.

  - **Default Data Loader**  
    - Type: LangChain Document Loader  
    - Role: Loads PDF binary data and splits pages for processing.  
    - Configuration: Uses pdfLoader with page splitting enabled.  
    - Inputs: Recursive Character Text Splitter output.  
    - Outputs: Document chunks.  
    - Edge Cases: Corrupted PDFs.

  - **Recursive Character Text Splitter**  
    - Type: LangChain Text Splitter  
    - Role: Splits text into chunks with 100-character overlap for embedding.  
    - Inputs: Default Data Loader output.  
    - Outputs: Text chunks for embedding.  
    - Edge Cases: Very small documents.

  - **Embeddings Ollama**  
    - Type: LangChain Embeddings  
    - Role: Generates vector embeddings for text chunks using Ollama API model `nomic-embed-text:latest`.  
    - Inputs: Text chunks.  
    - Outputs: Embeddings.  
    - Edge Cases: API rate limits, embedding failures.

  - **Qdrant Vector Store**  
    - Type: LangChain Vector Store  
    - Role: Inserts embeddings into Qdrant collection named "bitrix-docs".  
    - Inputs: Embeddings Ollama output.  
    - Outputs: Confirmation of insertion.  
    - Edge Cases: Database connectivity issues.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Documentation comment explaining the subworkflow for bot registration and vector storage of files.  
    - Content: "Subworkflow for Register Bot. Here are files vector stored for Open line chanel bot. After files are stored they are moved to subfolder."  
    - Inputs/Outputs: None.

---

#### 2.5 Subworkflow Execution & Parameter Merging

- **Overview:**  
  Supports modular execution of subworkflows with parameter merging to enable reuse and maintainability.

- **Nodes Involved:**  
  - Merge parameters for Subworkflow (Merge)  
  - Execute subworkflow (Execute Workflow)  
  - Success Response (Respond to Webhook)

- **Node Details:**

  - **Merge parameters for Subworkflow**  
    - Type: Merge  
    - Role: Combines multiple input parameters into a single object for subworkflow input.  
    - Configuration: Combine mode set to combine all inputs.  
    - Inputs: Various nodes (e.g., Register Bot).  
    - Outputs: Merged parameters for subworkflow execution.

  - **Execute subworkflow**  
    - Type: Execute Workflow  
    - Role: Executes the same workflow or another workflow by ID with merged parameters.  
    - Configuration: Uses current workflow ID for recursive or modular execution.  
    - Inputs: Merge parameters output.  
    - Outputs: Success Response.  
    - Edge Cases: Recursive loops, parameter mismatches.

  - **Success Response**  
    - As described previously.

---

### 3. Summary Table

| Node Name                          | Node Type                      | Functional Role                                   | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                  |
|-----------------------------------|--------------------------------|-------------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Bitrix24 Handler                  | Webhook                       | Entry point for Bitrix24 Open Line events       | External HTTP request            | Credentials                       |                                                                                              |
| Credentials                      | Set                           | Extracts and sets authentication and config     | Bitrix24 Handler                | Validate Token                   |                                                                                              |
| Validate Token                   | If                            | Validates application token                      | Credentials                     | Route Event, Error Response       |                                                                                              |
| Error Response                  | Respond to Webhook             | Returns 401 error for invalid token             | Validate Token                  | -                                 |                                                                                              |
| Route Event                    | Switch                        | Routes events by type                            | Validate Token                  | Process Message, Process Join, Process Install, Process Delete |                                                                                              |
| Process Message                | Function                      | Extracts message and session info                | Route Event                    | Question and Answer Chain         |                                                                                              |
| Process Join                   | Function                      | Prepares welcome menu message                     | Route Event                    | Send Join Message                |                                                                                              |
| Process Install                | Function                      | Prepares bot registration parameters             | Route Event                    | Register Bot                    |                                                                                              |
| Process Delete                | NoOp                          | Placeholder for bot deletion event                | Route Event                    | Success Response                |                                                                                              |
| Send Join Message             | HTTP Request                  | Sends welcome message to Bitrix24 chat           | Process Join                   | Success Response                |                                                                                              |
| Register Bot                  | HTTP Request                  | Registers bot with Bitrix24                        | Process Install                | Success Response, Merge parameters for Subworkflow |                                                                                              |
| Success Response             | Respond to Webhook             | Returns 200 success response                      | Send Join Message, Register Bot, Send Message, Process Delete, Execute subworkflow | -                                 |                                                                                              |
| Question and Answer Chain    | LangChain Retrieval QA Chain   | Generates AI response using retrieved context    | Process Message                | Prepare output parameters        |                                                                                              |
| Vector Store Retriever       | LangChain Retriever            | Retrieves relevant document chunks from Qdrant  | Qdrant Vector Store1           | Question and Answer Chain         |                                                                                              |
| Google Gemini Chat Model     | LangChain LLM Chat Model       | Generates natural language response               | Question and Answer Chain      | Question and Answer Chain         |                                                                                              |
| Prepare output parameters    | Set                           | Cleans and prepares AI response for sending      | Question and Answer Chain      | Send Message                    |                                                                                              |
| Send Message                | HTTP Request                  | Sends AI response to Bitrix24 chat                | Prepare output parameters      | Success Response                |                                                                                              |
| Execute Workflow Trigger    | Execute Workflow Trigger       | Initiates document processing                      | Manual or scheduled trigger    | Get a list of available storages |                                                                                              |
| Get a list of available storages | HTTP Request                  | Retrieves Bitrix24 storage list                    | Execute Workflow Trigger       | Get a list of List of Files and Folders |                                                                                              |
| Get a list of List of Files and Folders | HTTP Request                  | Retrieves folders filtered by name                 | Get a list of available storages | Get a list of Folders files       |                                                                                              |
| Get a list of Folders files  | HTTP Request                  | Retrieves files inside folder                       | Get a list of List of Files and Folders | Split Out folder files and folders |                                                                                              |
| Split Out folder files and folders | SplitOut                      | Splits folder content list into individual items  | Get a list of Folders files    | Filter for files                |                                                                                              |
| Filter for files             | Filter                        | Filters only files from folder items               | Split Out folder files and folders | Download file                  |                                                                                              |
| Download file               | HTTP Request                  | Downloads file binary data                          | Filter for files               | Move files to Vector stored folder, Qdrant Vector Store |                                                                                              |
| Move files to Vector stored folder | HTTP Request                  | Moves processed files to designated subfolder      | Download file                 | -                             |                                                                                              |
| Default Data Loader          | LangChain Document Loader      | Loads and splits PDF documents                      | Recursive Character Text Splitter | Qdrant Vector Store            |                                                                                              |
| Recursive Character Text Splitter | LangChain Text Splitter        | Splits text into overlapping chunks                 | Default Data Loader            | Embeddings Ollama              |                                                                                              |
| Embeddings Ollama            | LangChain Embeddings           | Generates vector embeddings for text chunks        | Recursive Character Text Splitter | Qdrant Vector Store            |                                                                                              |
| Qdrant Vector Store          | LangChain Vector Store         | Inserts embeddings into Qdrant collection           | Embeddings Ollama             | Vector Store Retriever (for retrieval), Confirmation |                                                                                              |
| Sticky Note                 | Sticky Note                   | Documentation comment                               | -                              | -                               | Subworkflow for Register Bot. Here are files vector stored for Open line chanel bot. After files are stored they are moved to subfolder. |
| Merge parameters for Subworkflow | Merge                         | Combines parameters for subworkflow execution       | Register Bot                  | Execute subworkflow            |                                                                                              |
| Execute subworkflow          | Execute Workflow              | Executes subworkflow with merged parameters          | Merge parameters for Subworkflow | Success Response              |                                                                                              |
| Qdrant Vector Store1         | LangChain Vector Store         | Vector store for retrieval operations                | Embeddings Ollama1            | Vector Store Retriever          |                                                                                              |
| Embeddings Ollama1           | LangChain Embeddings           | Embeddings generation for retrieval                   | Recursive Character Text Splitter | Qdrant Vector Store1          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `bitrix24/openchannel-rag-bothandler.php`  
   - Response Mode: Response Node

2. **Create Credentials Node (Set)**  
   - Assign variables:  
     - CLIENT_ID: static string (e.g., `local.67c8f9e81cb353.30162021`)  
     - CLIENT_SECRET: static string  
     - application_token: from incoming JSON `body['auth[application_token]']`  
     - domain: from incoming JSON `body['auth[domain]']`  
     - access_token: from incoming JSON `body['auth[access_token]']`  
     - storageName: `"Shared drive"`  
     - folderName: `"Open line chat bot documents"`

3. **Create Validate Token Node (If)**  
   - Condition:  
     - CLIENT_ID equals application_token OR always true fallback  
   - True output: Route Event  
   - False output: Error Response

4. **Create Error Response Node**  
   - Respond with HTTP 401  
   - JSON body: `{ "result": false, "error": "Invalid application token" }`

5. **Create Route Event Node (Switch)**  
   - Route based on `body.event` field:  
     - ONIMBOTMESSAGEADD → Process Message  
     - ONIMBOTJOINCHAT → Process Join  
     - ONAPPINSTALL → Process Install  
     - ONIMBOTDELETE → Process Delete

6. **Create Process Message Node (Function)**  
   - Extract message, dialog ID, session ID, bot ID, user ID, auth tokens from nested JSON fields  
   - Return structured JSON with these fields plus original message and a simple echo response for testing

7. **Create Process Join Node (Function)**  
   - Extract dialog ID and auth tokens  
   - Return a fixed menu message string with options

8. **Create Process Install Node (Function)**  
   - Extract webhook URL and auth tokens  
   - Prepare bot registration parameters including bot code, type, event URLs, and bot properties

9. **Create Process Delete Node (NoOp)**  
   - Placeholder node, no operation

10. **Create Send Join Message Node (HTTP Request)**  
    - POST to `https://{DOMAIN}/rest/imbot.message.add?auth={AUTH}`  
    - Body parameters: DIALOG_ID, MESSAGE, AUTH

11. **Create Register Bot Node (HTTP Request)**  
    - POST to `https://{DOMAIN}/rest/imbot.register?auth={AUTH}`  
    - Body parameters: CODE, TYPE, EVENT_MESSAGE_ADD, EVENT_WELCOME_MESSAGE, EVENT_BOT_DELETE, PROPERTIES, CLIENT_ID, CLIENT_SECRET, OPENLINE

12. **Create Success Response Node (Respond to Webhook)**  
    - HTTP 200  
    - JSON body: `{ "result": true }`

13. **Create Question and Answer Chain Node (LangChain Retrieval QA)**  
    - Input text: `MESSAGE_ORI`  
    - System prompt: instruct to answer only with keys DIALOG_ID, AUTH, DOMAIN, MESSAGE  
    - Special response for "find out more about me"

14. **Create Vector Store Retriever Node (LangChain Retriever)**  
    - topK: 10  
    - Connect to Qdrant Vector Store1

15. **Create Google Gemini Chat Model Node (LangChain LLM Chat Model)**  
    - Model name: `models/gemini-2.0-flash`

16. **Create Prepare output parameters Node (Set)**  
    - Clean AI response text by removing markdown and trailing backticks  
    - Assign to `data` object

17. **Create Send Message Node (HTTP Request)**  
    - POST to `https://{DOMAIN}/rest/imbot.message.add?auth={AUTH}`  
    - Body parameters: DIALOG_ID, MESSAGE, AUTH

18. **Create Document Processing Trigger Node (Execute Workflow Trigger)**  
    - Manual or scheduled trigger to start document processing

19. **Create Get a list of available storages Node (HTTP Request)**  
    - POST to `https://{domain}/rest/disk.storage.getlist.json?auth={access_token}`  
    - Filter by ENTITY_TYPE = "common" and NAME = storageName

20. **Create Get a list of List of Files and Folders Node (HTTP Request)**  
    - POST to `https://{domain}/rest/disk.storage.getchildren.json?auth={access_token}`  
    - Filter by TYPE = "folder" and NAME = folderName

21. **Create Get a list of Folders files Node (HTTP Request)**  
    - POST to `https://{domain}/rest/disk.folder.getchildren.json?auth={access_token}`  
    - Parameter: folder ID from previous step

22. **Create Split Out folder files and folders Node (SplitOut)**  
    - Field to split: `result`

23. **Create Filter for files Node (Filter)**  
    - Condition: TYPE equals "file"

24. **Create Download file Node (HTTP Request)**  
    - URL: file download URL from JSON  
    - Method: GET

25. **Create Move files to Vector stored folder Node (HTTP Request)**  
    - POST to `https://{domain}/rest/disk.file.moveto.json?auth={access_token}`  
    - Parameters: file ID, target folder ID

26. **Create Default Data Loader Node (LangChain Document Loader)**  
    - Loader: pdfLoader  
    - Option: splitPages = true  
    - Input: binary PDF data

27. **Create Recursive Character Text Splitter Node (LangChain Text Splitter)**  
    - Chunk overlap: 100 characters

28. **Create Embeddings Ollama Node (LangChain Embeddings)**  
    - Model: `nomic-embed-text:latest`

29. **Create Qdrant Vector Store Node (LangChain Vector Store)**  
    - Mode: insert  
    - Collection: "bitrix-docs"

30. **Create Sticky Note Node**  
    - Content: "Subworkflow for Register Bot. Here are files vector stored for Open line chanel bot. After files are stored they are moved to subfolder."

31. **Create Merge parameters for Subworkflow Node (Merge)**  
    - Mode: combine all inputs

32. **Create Execute subworkflow Node (Execute Workflow)**  
    - Workflow ID: current workflow ID or target subworkflow ID

33. **Connect nodes according to described flow**  
    - Bitrix24 Handler → Credentials → Validate Token → Route Event → respective processing nodes  
    - Process Message → Question and Answer Chain → Prepare output parameters → Send Message → Success Response  
    - Process Join → Send Join Message → Success Response  
    - Process Install → Register Bot → Merge parameters → Execute subworkflow → Success Response  
    - Document processing nodes connected sequentially from trigger to vector store insertion

34. **Configure credentials for:**  
    - Bitrix24 API (OAuth2 or token-based)  
    - Ollama API (embedding generation)  
    - Qdrant API (vector database)  
    - Google Gemini API (LLM chat model)

35. **Set default values and constraints:**  
    - Folder names and storage names as per Bitrix24 setup  
    - API keys and tokens securely stored in n8n credentials  
    - Ensure webhook URLs are correctly configured in Bitrix24 app

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow demonstrates integration of Bitrix24 Open Line channels with advanced AI-powered RAG chatbot capabilities. | Workflow description and use cases provided in the initial overview.                                      |
| The vector store used is Qdrant, enabling semantic search beyond keyword matching.                                      | Qdrant official site: https://qdrant.tech/                                                               |
| Embeddings are generated using Ollama API with the model `nomic-embed-text:latest`.                                     | Ollama API documentation: https://ollama.com/docs                                                        |
| AI responses are generated using Google Gemini model `models/gemini-2.0-flash`.                                         | Google Gemini info: https://developers.google.com/ai/gemini                                               |
| Bitrix24 API endpoints used include `imbot.register` and `imbot.message.add` for bot registration and messaging.       | Bitrix24 REST API docs: https://training.bitrix24.com/rest_help/rest_sum.php                              |
| The system prompt in the QA chain enforces strict JSON key-value response format to maintain integration consistency.  | Prompt template is embedded in the Question and Answer Chain node configuration.                          |
| The workflow includes a modular subworkflow execution pattern for scalability and maintainability.                      | Subworkflow node executes the same or other workflows with merged parameters.                             |
| Sticky notes in the workflow provide contextual documentation for developers and maintainers.                          | Sticky note content is included in the summary table for reference.                                      |

---

This structured documentation provides a comprehensive understanding of the Bitrix24 AI-Powered RAG Chatbot workflow, enabling advanced users and AI agents to analyze, reproduce, and extend the workflow confidently.