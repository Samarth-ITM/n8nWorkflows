Build an AI Powered Phone Agent 📞🤖 with Retell, Google Calendar and RAG

https://n8nworkflows.xyz/workflows/build-an-ai-powered-phone-agent------with-retell--google-calendar-and-rag-3563


# Build an AI Powered Phone Agent 📞🤖 with Retell, Google Calendar and RAG

### 1. Workflow Overview

This workflow implements an AI-powered phone agent using RetellAI, Google Calendar, and a Retrieval-Augmented Generation (RAG) system with Qdrant vector store. It serves two primary functions:

- **1.1 Appointment Booking:**  
  Captures call events, extracts and summarizes call transcripts, parses structured data (names, contact info, dates), converts dates to Google Calendar format, and creates calendar events automatically.

- **1.2 RAG-based Information Retrieval:**  
  Processes user queries by retrieving relevant documents from a Qdrant vector store, uses an AI agent to generate context-aware answers, and responds via webhook.

The workflow also includes setup and maintenance blocks for managing the Qdrant collection and document vectorization from Google Drive.

Logical blocks are:

- **Block 1: Call Event Reception and Processing** (Webhook, Filtering, Data Extraction, Summarization)  
- **Block 2: Appointment Booking** (Date conversion, Calendar event creation)  
- **Block 3: RAG Query Handling** (Webhook, Vector retrieval, AI agent response)  
- **Block 4: Qdrant Collection Setup and Document Vectorization** (HTTP requests, Google Drive integration, embeddings)  
- **Block 5: Notifications and Monitoring** (Telegram alerts)  
- **Block 6: Setup Instructions and Documentation** (Sticky notes)

---

### 2. Block-by-Block Analysis

#### Block 1: Call Event Reception and Processing

- **Overview:**  
  Receives call events from RetellAI webhook, filters relevant events (`call_ended` or `call_analyzed`), extracts call details, summarizes the transcript using OpenAI, and parses structured output.

- **Nodes Involved:**  
  - `n8n_call` (Webhook)  
  - `Filter` (Filter)  
  - `Set call fields` (Set)  
  - `OpenAI Chat Model` (OpenAI GPT-4o-mini)  
  - `Structured Output Parser` (Structured parser for extracted fields)  
  - `Extract key points` (LLM chain for summarization)  
  - `Telegram` (Notification)

- **Node Details:**

  - **n8n_call**  
    - Type: Webhook  
    - Role: Entry point for call events from RetellAI  
    - Config: POST method, path set to unique webhook ID  
    - Inputs: External HTTP POST from RetellAI  
    - Outputs: JSON with call event data  
    - Edge cases: Missing or malformed webhook payloads, unauthorized calls

  - **Filter**  
    - Type: Filter  
    - Role: Passes only `call_ended` or `call_analyzed` events  
    - Config: Checks `$json.body.event` equals either event type  
    - Inputs: From `n8n_call`  
    - Outputs: Passes filtered events downstream  
    - Edge cases: Unexpected event types, empty event field

  - **Set call fields**  
    - Type: Set  
    - Role: Extracts and assigns key call details (transcript, duration, caller info, recording URL, etc.) into named fields for downstream use  
    - Config: Uses expressions to map nested JSON fields from webhook payload  
    - Inputs: From `Filter`  
    - Outputs: JSON with simplified call data fields  
    - Edge cases: Missing fields in webhook payload, expression evaluation errors

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model (GPT-4o-mini)  
    - Role: Summarizes call transcript into key points  
    - Config: Uses a custom prompt instructing the AI to analyze transcript and extract key points and action items  
    - Inputs: Call transcript and metadata from `Set call fields`  
    - Outputs: AI-generated summary text  
    - Edge cases: API rate limits, timeout, incomplete transcript

  - **Structured Output Parser**  
    - Type: Langchain structured output parser  
    - Role: Parses AI summary into structured JSON fields (first_name, last_name, email, telephone, summary, date, dateTime)  
    - Config: Manual JSON schema defining expected fields and types  
    - Inputs: AI summary text from `OpenAI Chat Model`  
    - Outputs: Parsed structured data for calendar booking and notifications  
    - Edge cases: Parsing failures if AI output deviates from schema

  - **Extract key points**  
    - Type: Langchain chain LLM  
    - Role: Further processes transcript to create concise, structured summaries with action items  
    - Config: Custom prompt emphasizing objectivity and accuracy  
    - Inputs: Data from `Set call fields`  
    - Outputs: Summarized key points  
    - Edge cases: Similar to OpenAI node; depends on model response quality

  - **Telegram**  
    - Type: Telegram node  
    - Role: Sends call summary and extracted details as a message to a configured Telegram chat  
    - Config: Uses expressions to format message with structured output fields  
    - Inputs: From `Extract key points`  
    - Outputs: Telegram message sent  
    - Edge cases: Invalid chat ID, Telegram API errors

---

#### Block 2: Appointment Booking

- **Overview:**  
  Converts extracted date/time from call data into Google Calendar-compatible format and creates calendar events.

- **Nodes Involved:**  
  - `n8n_check_available` (Webhook)  
  - `Concert start date` (LLM chain for date conversion)  
  - `Structured Output Parser1` (Parses date conversion output)  
  - `Google Calendar` (Event creation)  

- **Node Details:**

  - **n8n_check_available**  
    - Type: Webhook  
    - Role: Receives booking requests (e.g., from RetellAI function node)  
    - Config: POST method, unique webhook path, responds with last node output  
    - Inputs: External HTTP POST with booking data (including date)  
    - Outputs: Passes booking data downstream  
    - Edge cases: Invalid or missing date fields, unauthorized access

  - **Concert start date**  
    - Type: Langchain chain LLM  
    - Role: Converts user-provided date to Google Calendar start and end date/time (adds 1 hour to end)  
    - Config: Custom prompt to parse and format date strings  
    - Inputs: Date string from webhook payload  
    - Outputs: JSON with `start` and `end` date strings  
    - Edge cases: Ambiguous or invalid date formats, parsing errors

  - **Structured Output Parser1**  
    - Type: Langchain structured output parser  
    - Role: Parses date conversion output into structured JSON with `start` and `end` fields  
    - Config: Manual schema with string types for start and end  
    - Inputs: Text output from `Concert start date`  
    - Outputs: Structured date/time fields for calendar node  
    - Edge cases: Parsing failures if output format deviates

  - **Google Calendar**  
    - Type: Google Calendar node  
    - Role: Creates calendar event with parsed start/end times and event details  
    - Config: Uses OAuth2 credentials, calendar email set, summary and description fields configurable  
    - Inputs: Structured date/time and event info from parser  
    - Outputs: Event creation confirmation  
    - Edge cases: API quota limits, invalid calendar ID, authentication errors

---

#### Block 3: RAG Query Handling

- **Overview:**  
  Handles incoming RAG queries via webhook, retrieves relevant documents from Qdrant vector store, and generates AI responses using the knowledge base.

- **Nodes Involved:**  
  - `n8n_rag_function` (Webhook)  
  - `Retrive Qdrant Vector Store` (Vector retrieval)  
  - `Embeddings OpenAI2` (Embeddings generation)  
  - `RAG` (Vector store tool)  
  - `OpenAI Chat Model2` (AI agent)  
  - `Retrive Agent` (Conversational AI agent)  
  - `Respond to Webhook` (Webhook response)

- **Node Details:**

  - **n8n_rag_function**  
    - Type: Webhook  
    - Role: Entry point for RAG queries from RetellAI or other sources  
    - Config: POST method, unique webhook path, response mode set to respond via node  
    - Inputs: Query text and metadata in POST body  
    - Outputs: Passes query downstream  
    - Edge cases: Missing query text, unauthorized requests

  - **Embeddings OpenAI2**  
    - Type: OpenAI embeddings node  
    - Role: Generates vector embeddings for query text to search vector store  
    - Config: Uses OpenAI API with default parameters  
    - Inputs: Query text from webhook  
    - Outputs: Embedding vector for retrieval  
    - Edge cases: API limits, embedding failures

  - **Retrive Qdrant Vector Store**  
    - Type: Qdrant vector store retrieval node  
    - Role: Searches Qdrant collection for documents similar to query embedding  
    - Config: Collection name set (e.g., "scarperia"), uses Qdrant API credentials  
    - Inputs: Embedding vector  
    - Outputs: Retrieved documents relevant to query  
    - Edge cases: Connection errors, empty results

  - **RAG**  
    - Type: Langchain vector store tool  
    - Role: Combines retrieved documents with AI model to generate context-aware answers  
    - Config: Named "company_data" with description for company knowledge retrieval  
    - Inputs: Retrieved documents  
    - Outputs: Contextualized AI response  
    - Edge cases: Model errors, incomplete data

  - **OpenAI Chat Model2**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Processes query and retrieved data to generate final answer  
    - Config: GPT-4o-mini model, OpenAI credentials  
    - Inputs: Query and documents from RAG node  
    - Outputs: AI-generated response text  
    - Edge cases: API errors, response timeouts

  - **Retrive Agent**  
    - Type: Langchain conversational agent  
    - Role: Acts as AI assistant specialized for electronics store, answering in Italian with professional tone  
    - Config: System message defines behavior, uses retrieved knowledge base  
    - Inputs: Query text from webhook  
    - Outputs: Final answer to be sent back  
    - Edge cases: Out-of-scope questions, fallback handling

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends AI-generated response back to the requester synchronously  
    - Inputs: From `Retrive Agent`  
    - Outputs: HTTP response to webhook caller  
    - Edge cases: Response formatting errors, timeout

---

#### Block 4: Qdrant Collection Setup and Document Vectorization

- **Overview:**  
  Manages Qdrant collection lifecycle and document ingestion from Google Drive, including vectorization with OpenAI embeddings.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual trigger)  
  - `Create collection` (HTTP request)  
  - `Refresh collection` (HTTP request to delete points)  
  - `Get folder` (Google Drive)  
  - `Download Files` (Google Drive)  
  - `Embeddings OpenAI` (OpenAI embeddings)  
  - `Default Data Loader` (Document loader)  
  - `Token Splitter` (Text splitter)  
  - `Qdrant Vector Store` (Vector store insert)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual trigger  
    - Role: Starts collection creation and refresh process manually  
    - Inputs: None  
    - Outputs: Triggers HTTP requests

  - **Create collection**  
    - Type: HTTP Request  
    - Role: Creates a new Qdrant collection via API  
    - Config: POST to `https://QDRANTURL/collections/COLLECTION` with empty filter body, uses HTTP header auth  
    - Inputs: Trigger from manual node  
    - Outputs: API response confirming collection creation  
    - Edge cases: API errors, invalid credentials

  - **Refresh collection**  
    - Type: HTTP Request  
    - Role: Deletes all points in Qdrant collection to refresh data  
    - Config: POST to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter body  
    - Inputs: Trigger from manual node  
    - Outputs: API response confirming deletion  
    - Edge cases: API errors, partial deletions

  - **Get folder**  
    - Type: Google Drive node  
    - Role: Lists files in specified Google Drive folder for ingestion  
    - Config: Folder ID set via expression, drive ID "My Drive"  
    - Inputs: Trigger from `Refresh collection`  
    - Outputs: List of files metadata  
    - Edge cases: Permission errors, empty folder

  - **Download Files**  
    - Type: Google Drive node  
    - Role: Downloads each file as plain text for processing  
    - Config: Converts Google Docs to text/plain format  
    - Inputs: File IDs from `Get folder`  
    - Outputs: Binary file content  
    - Edge cases: Download failures, unsupported file types

  - **Embeddings OpenAI**  
    - Type: OpenAI embeddings node  
    - Role: Generates embeddings for downloaded documents  
    - Config: Uses OpenAI API  
    - Inputs: Binary file content  
    - Outputs: Embedding vectors  
    - Edge cases: API limits, file size constraints

  - **Default Data Loader**  
    - Type: Langchain document loader  
    - Role: Loads binary data into document format for vector store  
    - Config: Data type set to binary  
    - Inputs: Binary data from `Download Files`  
    - Outputs: Document objects  
    - Edge cases: Data format errors

  - **Token Splitter**  
    - Type: Langchain text splitter  
    - Role: Splits documents into chunks of 300 tokens with 30 token overlap for better vectorization  
    - Config: Chunk size 300, overlap 30  
    - Inputs: Documents from loader  
    - Outputs: Text chunks  
    - Edge cases: Improper splitting if document is too short

  - **Qdrant Vector Store**  
    - Type: Langchain vector store insert  
    - Role: Inserts vectorized document chunks into Qdrant collection  
    - Config: Collection ID set, uses Qdrant API credentials  
    - Inputs: Embeddings and chunks  
    - Outputs: Confirmation of insertion  
    - Edge cases: API errors, partial inserts

---

#### Block 5: Notifications and Monitoring

- **Overview:**  
  Sends call summaries and extracted data to Telegram for monitoring and alerts.

- **Nodes Involved:**  
  - `Telegram`

- **Node Details:**  
  - See Telegram node in Block 1.

---

#### Block 6: Setup Instructions and Documentation

- **Overview:**  
  Provides detailed setup instructions and guidance via sticky notes embedded in the workflow.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note2`  
  - `Sticky Note3`  
  - `Sticky Note4`  
  - `Sticky Note5`  
  - `Sticky Note6`

- **Node Details:**  
  - Sticky notes contain step-by-step instructions for RetellAI agent setup, Qdrant collection creation, document vectorization, Telegram chat ID setup, calendar event creation, and general project overview.  
  - Includes links to RetellAI and images for visual guidance.

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                       |
|-------------------------|----------------------------------------|----------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| n8n_call                | Webhook                                | Receives call events from RetellAI     |                             | Filter                      | # STEP 4 Intercept the "end call" event and get the full call transcript - Add your CHAT_ID in Telegram node |
| Filter                  | Filter                                 | Filters relevant call events            | n8n_call                    | Set call fields             | # STEP 4 Intercept the "end call" event and get the full call transcript - Add your CHAT_ID in Telegram node |
| Set call fields          | Set                                    | Extracts key call details               | Filter                      | Extract key points          |                                                                                                 |
| OpenAI Chat Model        | Langchain OpenAI Chat Model (GPT-4o-mini) | Summarizes call transcript              | Set call fields             | Structured Output Parser    |                                                                                                 |
| Structured Output Parser | Langchain Structured Output Parser     | Parses AI summary into structured data | OpenAI Chat Model           | Extract key points          |                                                                                                 |
| Extract key points       | Langchain Chain LLM                    | Creates concise call summary            | Set call fields             | Telegram                    |                                                                                                 |
| Telegram                | Telegram                               | Sends call summary notification         | Extract key points          |                             | # STEP 4 Intercept the "end call" event and get the full call transcript - Add your CHAT_ID in Telegram node |
| n8n_check_available      | Webhook                                | Receives appointment booking requests   |                             | Concert start date          | # STEP 5 If required, create the event in the calendar - Enter the title and description of the event |
| Concert start date       | Langchain Chain LLM                    | Converts date to Google Calendar format | n8n_check_available         | Structured Output Parser1   | # STEP 5 If required, create the event in the calendar - Enter the title and description of the event |
| Structured Output Parser1| Langchain Structured Output Parser     | Parses date conversion output            | Concert start date          | Google Calendar             | # STEP 5 If required, create the event in the calendar - Enter the title and description of the event |
| Google Calendar          | Google Calendar                        | Creates calendar event                   | Structured Output Parser1   |                             | # STEP 5 If required, create the event in the calendar - Enter the title and description of the event |
| n8n_rag_function         | Webhook                                | Receives RAG queries                     |                             | Retrive Agent               | # STEP 6 If required retrieve the informations by RAG system                                    |
| Embeddings OpenAI2       | Langchain OpenAI Embeddings            | Generates embeddings for RAG queries    | n8n_rag_function            | Retrive Qdrant Vector Store | # STEP 6 If required retrieve the informations by RAG system                                    |
| Retrive Qdrant Vector Store | Langchain Vector Store Qdrant         | Retrieves relevant documents             | Embeddings OpenAI2          | RAG                         | # STEP 6 If required retrieve the informations by RAG system                                    |
| RAG                     | Langchain Tool Vector Store            | Combines documents with AI model         | Retrive Qdrant Vector Store | Retrive Agent               | # STEP 6 If required retrieve the informations by RAG system                                    |
| OpenAI Chat Model2       | Langchain OpenAI Chat Model             | Processes query with retrieved data      | RAG                         | Retrive Agent               | # STEP 6 If required retrieve the informations by RAG system                                    |
| Retrive Agent           | Langchain Agent                        | AI assistant answering RAG queries       | n8n_rag_function, OpenAI Chat Model2 | Respond to Webhook          | # STEP 6 If required retrieve the informations by RAG system                                    |
| Respond to Webhook       | Respond to Webhook                    | Sends AI response back to caller         | Retrive Agent               |                             | # STEP 6 If required retrieve the informations by RAG system                                    |
| When clicking ‘Test workflow’ | Manual Trigger                      | Starts Qdrant collection setup           |                             | Create collection, Refresh collection | # STEP 1 Create Qdrant Collection - Change QDRANTURL and COLLECTION                             |
| Create collection        | HTTP Request                          | Creates Qdrant collection                 | When clicking ‘Test workflow’ | Refresh collection          | # STEP 1 Create Qdrant Collection - Change QDRANTURL and COLLECTION                             |
| Refresh collection       | HTTP Request                          | Deletes all points in Qdrant collection   | When clicking ‘Test workflow’ | Get folder                 | # STEP 1 Create Qdrant Collection - Change QDRANTURL and COLLECTION                             |
| Get folder               | Google Drive                         | Lists files in Google Drive folder        | Refresh collection          | Download Files              | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Download Files           | Google Drive                         | Downloads files as plain text              | Get folder                  | Embeddings OpenAI           | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Embeddings OpenAI        | Langchain OpenAI Embeddings            | Generates embeddings for documents        | Download Files              | Qdrant Vector Store         | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Default Data Loader      | Langchain Document Loader             | Loads binary data into document format    | Token Splitter              | Qdrant Vector Store         | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Token Splitter           | Langchain Text Splitter               | Splits documents into chunks               | Default Data Loader         |                           | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Qdrant Vector Store      | Langchain Vector Store Insert         | Inserts document vectors into Qdrant      | Embeddings OpenAI, Default Data Loader |                           | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Sticky Note              | Sticky Note                          | Setup instructions for Retell AI agent    |                             |                             | # STEP 3 - RETELL AI - Register on https://retellai.com, create agent, set webhook URLs, buy phone number, create flow |
| Sticky Note1             | Sticky Note                          | Reminder to intercept end call event and set Telegram CHAT_ID |                             |                             | # STEP 4 Intercept the "end call" event and get the full call transcript - Add your CHAT_ID in Telegram node |
| Sticky Note2             | Sticky Note                          | Reminder to create calendar event with title and description |                             |                             | # STEP 5 If required, create the event in the calendar - Enter the title and description of the event |
| Sticky Note3             | Sticky Note                          | Reminder to create Qdrant collection with URL and collection name changes |                             |                             | # STEP 1 Create Qdrant Collection - Change QDRANTURL and COLLECTION                             |
| Sticky Note4             | Sticky Note                          | Reminder about document vectorization with Qdrant and Google Drive |                             |                             | # STEP 2 Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Sticky Note5             | Sticky Note                          | Reminder about RAG system information retrieval |                             |                             | # STEP 6 If required retrieve the informations by RAG system                                    |
| Sticky Note6             | Sticky Note                          | General project overview and Retell AI introduction |                             |                             | # Create your first AI Phone Agent - Overview and advantages                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook for Call Events (`n8n_call`):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `b352dd49-d3b3-4e0a-a781-17137f7199c8`)  
   - Purpose: Receive call events from RetellAI agent webhook

2. **Add Filter Node (`Filter`):**  
   - Type: Filter  
   - Condition: Pass only if `$json.body.event` equals `call_ended` or `call_analyzed`  
   - Connect input from `n8n_call`

3. **Add Set Node to Extract Call Fields (`Set call fields`):**  
   - Type: Set  
   - Assign fields using expressions from webhook JSON: Transcript, Duration, From, To, Cost, Telephony Identifier, Disconnection reason, Recording URL, Public log URL  
   - Connect input from `Filter`

4. **Add OpenAI Chat Model Node (`OpenAI Chat Model`):**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API key  
   - Prompt: Summarize call transcript and extract key points and action items  
   - Connect input from `Set call fields`

5. **Add Structured Output Parser (`Structured Output Parser`):**  
   - Type: Langchain Structured Output Parser  
   - Schema: JSON schema defining fields like first_name, last_name, email, telephone, summary, date, dateTime  
   - Connect input from `OpenAI Chat Model`

6. **Add Chain LLM Node for Extracting Key Points (`Extract key points`):**  
   - Type: Langchain Chain LLM  
   - Prompt: Detailed instructions to create concise, objective summary from transcript  
   - Connect input from `Set call fields`

7. **Add Telegram Node (`Telegram`):**  
   - Type: Telegram  
   - Credentials: Telegram Bot API with chat ID configured  
   - Message: Format with extracted summary and contact info fields  
   - Connect input from `Extract key points`

8. **Create Webhook for Appointment Booking (`n8n_check_available`):**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `4dcd68b1-91d3-40bc-8aa6-c681126752b2`)  
   - Purpose: Receive booking requests with date info

9. **Add Chain LLM Node to Convert Dates (`Concert start date`):**  
   - Type: Langchain Chain LLM  
   - Prompt: Convert input date string to Google Calendar start and end date/time (end = start + 1 hour)  
   - Connect input from `n8n_check_available`

10. **Add Structured Output Parser for Date Conversion (`Structured Output Parser1`):**  
    - Type: Langchain Structured Output Parser  
    - Schema: JSON with `start` and `end` string fields  
    - Connect input from `Concert start date`

11. **Add Google Calendar Node (`Google Calendar`):**  
    - Type: Google Calendar  
    - Credentials: OAuth2 with calendar access  
    - Calendar: Set calendar email (e.g., `info@n3w.it`)  
    - Start and End: Use parsed `start` and `end` from previous node  
    - Summary and Description: Set event title and description as needed  
    - Connect input from `Structured Output Parser1`

12. **Create Webhook for RAG Queries (`n8n_rag_function`):**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Path: Unique identifier (e.g., `edb1e894-1210-4902-a34f-a014bbdad8d8`)  
    - Response Mode: Respond via node

13. **Add OpenAI Embeddings Node for Query (`Embeddings OpenAI2`):**  
    - Type: Langchain OpenAI Embeddings  
    - Credentials: OpenAI API key  
    - Connect input from `n8n_rag_function`

14. **Add Qdrant Vector Store Retrieval Node (`Retrive Qdrant Vector Store`):**  
    - Type: Langchain Vector Store Qdrant  
    - Credentials: Qdrant API key  
    - Collection: Set to your collection name (e.g., "scarperia")  
    - Connect input from `Embeddings OpenAI2`

15. **Add RAG Tool Node (`RAG`):**  
    - Type: Langchain Tool Vector Store  
    - Name: "company_data"  
    - Description: "Retrieve data about company knowledge from vector store"  
    - Connect input from `Retrive Qdrant Vector Store`

16. **Add OpenAI Chat Model Node for RAG (`OpenAI Chat Model2`):**  
    - Type: Langchain OpenAI Chat Model  
    - Model: `gpt-4o-mini`  
    - Credentials: OpenAI API key  
    - Connect input from `RAG`

17. **Add Conversational Agent Node (`Retrive Agent`):**  
    - Type: Langchain Agent  
    - Agent: Conversational Agent  
    - System Message: Define assistant role, language (Italian), tone, and knowledge base usage  
    - Connect inputs from `n8n_rag_function` and `OpenAI Chat Model2`

18. **Add Respond to Webhook Node (`Respond to Webhook`):**  
    - Type: Respond to Webhook  
    - Connect input from `Retrive Agent`

19. **Set up Qdrant Collection Creation and Document Vectorization:**  
    - Manual Trigger Node (`When clicking ‘Test workflow’`)  
    - HTTP Request Node to create collection (`Create collection`)  
    - HTTP Request Node to refresh collection (`Refresh collection`)  
    - Google Drive Node to get folder files (`Get folder`)  
    - Google Drive Node to download files (`Download Files`)  
    - OpenAI Embeddings Node for documents (`Embeddings OpenAI`)  
    - Langchain Document Loader (`Default Data Loader`)  
    - Langchain Text Splitter (`Token Splitter`)  
    - Qdrant Vector Store Insert Node (`Qdrant Vector Store`)  
    - Connect nodes in sequence as per above description

20. **Add Sticky Notes for Setup Instructions:**  
    - Create sticky notes with setup steps, links, and reminders as per original workflow

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Register on [Retell AI](https://retellai.com) to create an AI phone agent with $10 free credits. Create an agent, configure voice & language, and set webhook URLs to n8n webhook nodes (`n8n_call` for call events, `n8n_rag_function` for RAG queries). Purchase a Twilio phone number with +1 prefix and link it to the agent for phone call handling.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Retell AI platform setup instructions                                                             |
| Use the Qdrant vector store to manage company knowledge base documents. Create and refresh collections via HTTP requests. Upload and vectorize documents from Google Drive using OpenAI embeddings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Qdrant vector store setup and document ingestion                                                   |
| Telegram node requires a valid `CHAT_ID` to send notifications. Replace placeholder `CHAT_ID` with your actual Telegram chat identifier.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Telegram bot setup and chat ID configuration                                                      |
| Google Calendar node requires OAuth2 credentials with calendar access. Set calendar email and configure event summary and description fields appropriately.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Google Calendar API and OAuth2 setup                                                              |
| The AI assistant in RAG block is configured to answer in Italian for an electronics store use case. Modify system message to adapt tone, language, and domain knowledge for your use case.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | AI assistant customization                                                                         |
| The workflow uses Langchain nodes for advanced AI prompt chaining, structured output parsing, and vector store integration, requiring n8n version supporting these nodes (n8n 0.201+ recommended).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | n8n version and Langchain node requirements                                                       |
| Workflow timezone is set to Europe/Rome. Adjust timezone in workflow settings if deploying in different regions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow timezone configuration                                                                   |
| ![Retell AI Flow Image](https://i.postimg.cc/brtBkgfH/Retellai-flow.png) Visual guide for Retell AI agent flow creation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Retell AI flow visual guide                                                                       |
| Contact for customization and support: [info@n3w.it](mailto:info@n3w.it), LinkedIn: [https://www.linkedin.com/in/davideboizza/](https://www.linkedin.com/in/davideboizza/)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Consulting and support contact                                                                     |

---

This document provides a complete, detailed reference for understanding, reproducing, and customizing the AI Phone Agent workflow integrating RetellAI, Google Calendar, and RAG with Qdrant. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with the workflow.