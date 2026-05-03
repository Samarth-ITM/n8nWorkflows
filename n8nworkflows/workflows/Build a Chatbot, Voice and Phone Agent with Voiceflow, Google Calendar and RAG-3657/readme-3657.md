Build a Chatbot, Voice and Phone Agent with Voiceflow, Google Calendar and RAG

https://n8nworkflows.xyz/workflows/build-a-chatbot--voice-and-phone-agent-with-voiceflow--google-calendar-and-rag-3657


# Build a Chatbot, Voice and Phone Agent with Voiceflow, Google Calendar and RAG

### 1. Workflow Overview

This workflow orchestrates a multi-channel conversational agent integrating **Voiceflow**, **Google Calendar**, **Qdrant vector database**, **OpenAI**, and an external **order tracking API**. It supports chat, voice, and phone interactions, enabling order tracking, appointment scheduling, and knowledge-based Q&A via a Retrieval-Augmented Generation (RAG) system.

The workflow is logically divided into these main blocks:

- **1.1 Webhook Input Reception**: Three webhook nodes receive requests from Voiceflow corresponding to order tracking, appointment booking, and general product/service questions (RAG).

- **1.2 Order Tracking Block**: Processes order-related queries by calling an external API and returning order status.

- **1.3 Appointment Scheduling Block**: Parses natural language appointment dates using OpenAI, formats dates for Google Calendar, creates calendar events, and confirms success.

- **1.4 RAG System Block**: Handles general questions by retrieving relevant documents from a Qdrant vector store (populated from Google Drive documents), embedding them with OpenAI, and generating context-aware answers using GPT-4.

- **1.5 Qdrant Collection Setup and Document Ingestion**: Initializes and refreshes the Qdrant collection, downloads documents from Google Drive, converts them to embeddings, and inserts them into Qdrant.

- **1.6 Voiceflow Integration and Response Delivery**: Routes responses back to Voiceflow via webhook response nodes for multi-channel delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Input Reception

**Overview:**  
Receives incoming requests from Voiceflow for three distinct conversational intents: order tracking, appointment scheduling, and RAG-based queries.

**Nodes Involved:**  
- `n8n_order` (Webhook)  
- `n8n_appointment` (Webhook)  
- `n8n_rag` (Webhook)

**Node Details:**

- **n8n_order**  
  - Type: Webhook  
  - Role: Entry point for order tracking requests from Voiceflow.  
  - Config: Path set to unique webhook ID; response mode set to respond via downstream node.  
  - Inputs: External HTTP POST from Voiceflow.  
  - Outputs: Connects to `API URL Tracking`.  
  - Edge Cases: Invalid/missing order number or email; webhook timeout; malformed JSON.

- **n8n_appointment**  
  - Type: Webhook  
  - Role: Entry point for appointment booking requests.  
  - Config: Unique webhook path; response mode as above.  
  - Inputs: Voiceflow appointment data including natural language date.  
  - Outputs: Connects to `Concert start date`.  
  - Edge Cases: Invalid date formats; missing fields; webhook timeout.

- **n8n_rag**  
  - Type: Webhook  
  - Role: Entry point for general product/service questions routed to RAG system.  
  - Config: Unique webhook path; response mode as above.  
  - Inputs: User question text.  
  - Outputs: Connects to `Retrive Agent`.  
  - Edge Cases: Empty or irrelevant questions; webhook timeout.

---

#### 2.2 Order Tracking Block

**Overview:**  
Processes order tracking requests by querying an external API with order number and email, then formats and returns the order status.

**Nodes Involved:**  
- `API URL Tracking` (HTTP Request)  
- `Tracking response` (Set)  
- `Webhook tracking response` (Respond to Webhook)

**Node Details:**

- **API URL Tracking**  
  - Type: HTTP Request  
  - Role: Calls external order tracking API.  
  - Config: POST request to configured URL (`URL_TRACKING` placeholder), sending `Order_number` and `Email` from webhook JSON.  
  - Inputs: From `n8n_order`.  
  - Outputs: Order status JSON to `Tracking response`.  
  - Edge Cases: API authentication failure, network timeout, invalid order number, malformed response.

- **Tracking response**  
  - Type: Set  
  - Role: Formats API response into a user-friendly text string.  
  - Config: Sets `text` field to "Your order status is: {{ $json.status }}".  
  - Inputs: API response JSON.  
  - Outputs: To `Webhook tracking response`.  
  - Edge Cases: Missing `status` field in API response.

- **Webhook tracking response**  
  - Type: Respond to Webhook  
  - Role: Sends formatted order status back to Voiceflow.  
  - Inputs: From `Tracking response`.  
  - Outputs: HTTP response to Voiceflow webhook caller.  
  - Edge Cases: Response delivery failure.

---

#### 2.3 Appointment Scheduling Block

**Overview:**  
Converts natural language appointment dates to Google Calendar-compatible format using OpenAI, creates calendar events, and confirms success.

**Nodes Involved:**  
- `Concert start date` (Langchain LLM Chain)  
- `Structured Output Parser` (Output Parser Structured)  
- `Google Calendar` (Google Calendar node)  
- `Calendar response` (Set)  
- `Webhook calendar response` (Respond to Webhook)

**Node Details:**

- **Concert start date**  
  - Type: Langchain Chain LLM  
  - Role: Uses OpenAI GPT-4 mini model to convert natural language date to structured start and end date strings (end date = start + 1 hour).  
  - Config: Prompt instructs conversion and adding 1 hour; input is `Appointment_date` from webhook JSON.  
  - Inputs: From `n8n_appointment`.  
  - Outputs: JSON with `start` and `end` fields to `Structured Output Parser`.  
  - Edge Cases: Ambiguous or invalid date input; OpenAI API errors.

- **Structured Output Parser**  
  - Type: Output Parser Structured  
  - Role: Parses LLM output into JSON object with `start` and `end` strings.  
  - Config: JSON schema defines required fields.  
  - Inputs: From `Concert start date`.  
  - Outputs: Parsed JSON to `Google Calendar`.  
  - Edge Cases: Parsing failures if LLM output is malformed.

- **Google Calendar**  
  - Type: Google Calendar node  
  - Role: Creates calendar event using parsed start/end dates and email from appointment query.  
  - Config: Uses OAuth2 credentials; calendar set to `info@n3w.it`; summary includes email; description is static.  
  - Inputs: From `Structured Output Parser`.  
  - Outputs: To `Calendar response`.  
  - Edge Cases: Authentication errors; invalid date ranges; API quota limits.

- **Calendar response**  
  - Type: Set  
  - Role: Sets confirmation message text "L'evento è stato creato con successo" (Event created successfully).  
  - Inputs: From `Google Calendar`.  
  - Outputs: To `Webhook calendar response`.  
  - Edge Cases: None significant.

- **Webhook calendar response**  
  - Type: Respond to Webhook  
  - Role: Sends confirmation back to Voiceflow.  
  - Inputs: From `Calendar response`.  
  - Outputs: HTTP response to Voiceflow webhook caller.  
  - Edge Cases: Response delivery failure.

---

#### 2.4 RAG System Block

**Overview:**  
Answers general product/service questions by retrieving relevant documents from Qdrant vector store, embedding them with OpenAI, and generating context-aware answers using GPT-4.

**Nodes Involved:**  
- `Retrive Qdrant Vector Store` (Vector Store Qdrant)  
- `Embeddings OpenAI2` (Embeddings OpenAI)  
- `RAG` (Tool Vector Store)  
- `OpenAI Chat Model1` (Chat Model OpenAI)  
- `OpenAI Chat Model2` (Chat Model OpenAI)  
- `Retrive Agent` (Langchain Agent)  
- `n8n_rag` (Webhook)  
- `Webhook RAG response` (Respond to Webhook)

**Node Details:**

- **Retrive Qdrant Vector Store**  
  - Type: Vector Store Qdrant  
  - Role: Retrieves relevant documents from Qdrant collection `scarperia` based on query embeddings.  
  - Config: Uses Qdrant API credentials; collection name `scarperia`.  
  - Inputs: Embeddings from `Embeddings OpenAI2`.  
  - Outputs: To `RAG`.  
  - Edge Cases: Connection errors; empty results.

- **Embeddings OpenAI2**  
  - Type: Embeddings OpenAI  
  - Role: Generates embeddings for the user query text.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: From `Retrive Agent` (query text).  
  - Outputs: To `Retrive Qdrant Vector Store`.  
  - Edge Cases: API rate limits; invalid input.

- **RAG**  
  - Type: Tool Vector Store  
  - Role: Uses retrieved documents to augment the language model's context.  
  - Config: Named `company_data` with description for company knowledge retrieval.  
  - Inputs: From `OpenAI Chat Model1`.  
  - Outputs: To `OpenAI Chat Model2`.  
  - Edge Cases: Empty or irrelevant documents.

- **OpenAI Chat Model1**  
  - Type: Chat Model OpenAI  
  - Role: Generates initial language model output using GPT-4 mini.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: From `Retrive Qdrant Vector Store`.  
  - Outputs: To `RAG`.  
  - Edge Cases: API errors; model unavailability.

- **OpenAI Chat Model2**  
  - Type: Chat Model OpenAI  
  - Role: Finalizes response generation incorporating RAG context.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: From `RAG`.  
  - Outputs: To `Retrive Agent`.  
  - Edge Cases: API errors.

- **Retrive Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates the RAG pipeline, receives user question from webhook, manages system prompt for electronics store assistant, and outputs final answer.  
  - Config: System prompt defines assistant behavior, tone, and limitations; uses GPT-4 mini model.  
  - Inputs: From `n8n_rag`.  
  - Outputs: To `Webhook RAG response`.  
  - Edge Cases: Complex or out-of-scope questions; API failures.

- **Webhook RAG response**  
  - Type: Respond to Webhook  
  - Role: Sends generated answer back to Voiceflow.  
  - Inputs: From `Retrive Agent`.  
  - Outputs: HTTP response to Voiceflow webhook caller.  
  - Edge Cases: Response delivery failure.

---

#### 2.5 Qdrant Collection Setup and Document Ingestion

**Overview:**  
Initializes the Qdrant vector store collection, clears existing points, downloads documents from Google Drive, converts them to embeddings, and inserts them into Qdrant.

**Nodes Involved:**  
- `When clicking ‘Test workflow’` (Manual Trigger)  
- `Create collection` (HTTP Request)  
- `Refresh collection` (HTTP Request)  
- `Get folder` (Google Drive)  
- `Download Files` (Google Drive)  
- `Embeddings OpenAI` (Embeddings OpenAI)  
- `Default Data Loader` (Document Default Data Loader)  
- `Token Splitter` (Text Splitter Token Splitter)  
- `Qdrant Vector Store` (Vector Store Qdrant)  
- Sticky Notes: `Sticky Note3`, `Sticky Note4`

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the Qdrant collection setup and document ingestion process manually.  
  - Outputs: To `Create collection` and `Refresh collection`.  
  - Edge Cases: None.

- **Create collection**  
  - Type: HTTP Request  
  - Role: Creates a new Qdrant collection via API.  
  - Config: POST to `https://QDRANTURL/collections/COLLECTION` with empty filter; uses HTTP header auth with API key.  
  - Inputs: From manual trigger.  
  - Outputs: To `Refresh collection`.  
  - Edge Cases: API errors; invalid URL or credentials.

- **Refresh collection**  
  - Type: HTTP Request  
  - Role: Deletes all points in the Qdrant collection to refresh it.  
  - Config: POST to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter; same auth as above.  
  - Inputs: From `Create collection`.  
  - Outputs: To `Get folder`.  
  - Edge Cases: API errors.

- **Get folder**  
  - Type: Google Drive  
  - Role: Lists files in a specified Google Drive folder (`test-whatsapp`).  
  - Config: Uses Google Drive OAuth2 credentials; filters by folder ID.  
  - Inputs: From `Refresh collection`.  
  - Outputs: To `Download Files`.  
  - Edge Cases: Permission errors; empty folder.

- **Download Files**  
  - Type: Google Drive  
  - Role: Downloads each file from the folder, converting Google Docs to plain text.  
  - Config: File ID from previous node; conversion enabled for Google Docs to text/plain.  
  - Inputs: From `Get folder`.  
  - Outputs: To `Qdrant Vector Store` via document loader chain.  
  - Edge Cases: Download failures; unsupported file types.

- **Embeddings OpenAI**  
  - Type: Embeddings OpenAI  
  - Role: Generates embeddings for downloaded documents.  
  - Config: Uses OpenAI API credentials.  
  - Inputs: From `Default Data Loader`.  
  - Outputs: To `Qdrant Vector Store`.  
  - Edge Cases: API limits.

- **Default Data Loader**  
  - Type: Document Default Data Loader  
  - Role: Converts binary file data into document format for embedding.  
  - Inputs: From `Token Splitter`.  
  - Outputs: To `Embeddings OpenAI`.  
  - Edge Cases: Data format errors.

- **Token Splitter**  
  - Type: Text Splitter Token Splitter  
  - Role: Splits documents into chunks of 300 tokens with 30 token overlap for better embedding quality.  
  - Inputs: From `Default Data Loader`.  
  - Outputs: To `Default Data Loader`.  
  - Edge Cases: Chunking errors.

- **Qdrant Vector Store**  
  - Type: Vector Store Qdrant  
  - Role: Inserts document embeddings into the Qdrant collection.  
  - Config: Uses Qdrant API credentials; collection name configured dynamically.  
  - Inputs: From `Embeddings OpenAI` and `Download Files`.  
  - Outputs: None (end of ingestion pipeline).  
  - Edge Cases: API errors; insertion failures.

---

#### 2.6 Voiceflow Integration and Response Delivery

**Overview:**  
Routes responses from each processing block back to Voiceflow via webhook response nodes, enabling multi-channel delivery (chat, voice, phone).

**Nodes Involved:**  
- `Webhook tracking response`  
- `Webhook calendar response`  
- `Webhook RAG response`  
- Sticky Note: `Sticky Note`

**Node Details:**

- **Webhook tracking response**  
  - Sends order tracking status back to Voiceflow.

- **Webhook calendar response**  
  - Sends appointment confirmation back to Voiceflow.

- **Webhook RAG response**  
  - Sends RAG-generated answers back to Voiceflow.

- **Sticky Note**  
  - Provides instructions and links for Voiceflow setup, including webhook URL integration, project ID retrieval, widget installation, and optional Twilio phone agent setup.  
  - Contains multiple images illustrating Voiceflow configuration and agent deployment.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| n8n_order               | Webhook                          | Receives order tracking requests       | External (Voiceflow)          | API URL Tracking               | # STEP 3: If required retrieve info by Order system - Set your API URL Tracking service       |
| API URL Tracking        | HTTP Request                    | Calls external order tracking API      | n8n_order                    | Tracking response              | # STEP 3: If required retrieve info by Order system - Set your API URL Tracking service       |
| Tracking response       | Set                             | Formats order status response           | API URL Tracking             | Webhook tracking response     | # STEP 3: If required retrieve info by Order system - Set your API URL Tracking service       |
| Webhook tracking response | Respond to Webhook              | Sends order status back to Voiceflow   | Tracking response            | External (Voiceflow)           | # STEP 3: If required retrieve info by Order system - Set your API URL Tracking service       |
| n8n_appointment         | Webhook                          | Receives appointment booking requests  | External (Voiceflow)          | Concert start date            | # STEP 4: If required retrieve info by Appointment system                                   |
| Concert start date      | Langchain Chain LLM              | Converts natural language date          | n8n_appointment             | Structured Output Parser      | # STEP 4: If required retrieve info by Appointment system                                   |
| Structured Output Parser | Output Parser Structured         | Parses LLM output to JSON               | Concert start date           | Google Calendar              | # STEP 4: If required retrieve info by Appointment system                                   |
| Google Calendar         | Google Calendar                  | Creates calendar event                  | Structured Output Parser     | Calendar response             | # STEP 4: If required retrieve info by Appointment system                                   |
| Calendar response       | Set                             | Sets appointment confirmation message  | Google Calendar             | Webhook calendar response     | # STEP 4: If required retrieve info by Appointment system                                   |
| Webhook calendar response | Respond to Webhook              | Sends appointment confirmation to Voiceflow | Calendar response          | External (Voiceflow)           | # STEP 4: If required retrieve info by Appointment system                                   |
| n8n_rag                 | Webhook                          | Receives general questions for RAG     | External (Voiceflow)          | Retrive Agent                | # STEP 5: If required retrieve info by RAG system                                           |
| Retrive Agent           | Langchain Agent                 | Orchestrates RAG pipeline and answers  | n8n_rag                     | Webhook RAG response          | # STEP 5: If required retrieve info by RAG system                                           |
| Webhook RAG response    | Respond to Webhook              | Sends RAG answer back to Voiceflow     | Retrive Agent               | External (Voiceflow)           | # STEP 5: If required retrieve info by RAG system                                           |
| OpenAI Chat Model1      | Chat Model OpenAI               | Generates initial LLM output            | Retrive Qdrant Vector Store | RAG                          |                                                                                              |
| OpenAI Chat Model2      | Chat Model OpenAI               | Finalizes LLM response with context    | RAG                         | Retrive Agent                |                                                                                              |
| Retrive Qdrant Vector Store | Vector Store Qdrant           | Retrieves relevant documents            | Embeddings OpenAI2          | OpenAI Chat Model1            |                                                                                              |
| Embeddings OpenAI2      | Embeddings OpenAI              | Embeds user query text                  | Retrive Agent               | Retrive Qdrant Vector Store  |                                                                                              |
| When clicking ‘Test workflow’ | Manual Trigger               | Starts Qdrant collection setup          | -                            | Create collection, Refresh collection | # STEP 1: Create Qdrant Collection - Change QDRANTURL and COLLECTION                        |
| Create collection       | HTTP Request                    | Creates Qdrant collection               | When clicking ‘Test workflow’ | Refresh collection           | # STEP 1: Create Qdrant Collection - Change QDRANTURL and COLLECTION                        |
| Refresh collection      | HTTP Request                    | Clears Qdrant collection points         | Create collection           | Get folder                   | # STEP 1: Create Qdrant Collection - Change QDRANTURL and COLLECTION                        |
| Get folder              | Google Drive                   | Lists files in Google Drive folder      | Refresh collection          | Download Files               | # STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Download Files          | Google Drive                   | Downloads and converts files to text   | Get folder                  | Qdrant Vector Store          | # STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Embeddings OpenAI       | Embeddings OpenAI              | Embeds downloaded documents             | Default Data Loader         | Qdrant Vector Store          | # STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Default Data Loader     | Document Default Data Loader   | Converts binary data to document format | Token Splitter              | Embeddings OpenAI            | # STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Token Splitter          | Text Splitter Token Splitter   | Splits documents into chunks            | Default Data Loader         | Default Data Loader          | # STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Qdrant Vector Store     | Vector Store Qdrant            | Inserts embeddings into Qdrant          | Embeddings OpenAI, Download Files | -                      | # STEP 2: Documents vectorization with Qdrant and Google Drive - Change QDRANTURL and COLLECTION |
| Sticky Note             | Sticky Note                   | Voiceflow setup instructions and links | -                          | -                           | See detailed instructions and images for Voiceflow integration and Twilio phone agent setup |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes**  
   - Create three webhook nodes named: `n8n_order`, `n8n_appointment`, and `n8n_rag`.  
   - Configure each with unique webhook paths (e.g., `9ff7a394-5b4b-4790-a96b-c41c4ba27fa5` for order).  
   - Set response mode to "responseNode" to send responses from downstream nodes.

2. **Order Tracking Block**  
   - Add HTTP Request node `API URL Tracking`:  
     - Method: POST  
     - URL: Your order tracking API endpoint (`URL_TRACKING`)  
     - Body parameters: `Order_number` and `Email` from webhook JSON.  
     - Connect `n8n_order` → `API URL Tracking`.  
   - Add Set node `Tracking response`:  
     - Set field `text` to `"Your order status is: {{ $json.status }}"`.  
     - Connect `API URL Tracking` → `Tracking response`.  
   - Add Respond to Webhook node `Webhook tracking response`:  
     - Connect `Tracking response` → `Webhook tracking response`.  

3. **Appointment Scheduling Block**  
   - Add Langchain Chain LLM node `Concert start date`:  
     - Model: GPT-4 mini  
     - Prompt: Convert natural language date to Google Calendar format, add 1 hour for end date.  
     - Input: `Appointment_date` from webhook JSON.  
     - Connect `n8n_appointment` → `Concert start date`.  
   - Add Output Parser Structured node `Structured Output Parser`:  
     - Define JSON schema with `start` and `end` strings.  
     - Connect `Concert start date` → `Structured Output Parser`.  
   - Add Google Calendar node `Google Calendar`:  
     - Authenticate with Google Calendar OAuth2.  
     - Calendar: Select your calendar email (e.g., `info@n3w.it`).  
     - Start and End: Use expressions `{{$json.output.start}}` and `{{$json.output.end}}`.  
     - Summary: Include email from appointment query.  
     - Connect `Structured Output Parser` → `Google Calendar`.  
   - Add Set node `Calendar response`:  
     - Set field `text` to `"L'evento è stato creato con successo"`.  
     - Connect `Google Calendar` → `Calendar response`.  
   - Add Respond to Webhook node `Webhook calendar response`:  
     - Connect `Calendar response` → `Webhook calendar response`.  

4. **RAG System Block**  
   - Add Langchain Agent node `Retrive Agent`:  
     - Model: GPT-4 mini  
     - System prompt: Use provided detailed assistant prompt for electronics store.  
     - Input text: `Question` from webhook JSON.  
     - Connect `n8n_rag` → `Retrive Agent`.  
   - Add Embeddings OpenAI node `Embeddings OpenAI2`:  
     - Authenticate with OpenAI API.  
     - Connect `Retrive Agent` → `Embeddings OpenAI2`.  
   - Add Vector Store Qdrant node `Retrive Qdrant Vector Store`:  
     - Authenticate with Qdrant API.  
     - Collection: `scarperia`.  
     - Connect `Embeddings OpenAI2` → `Retrive Qdrant Vector Store`.  
   - Add Chat Model OpenAI nodes `OpenAI Chat Model1` and `OpenAI Chat Model2`:  
     - Both use GPT-4 mini and OpenAI credentials.  
     - Connect `Retrive Qdrant Vector Store` → `OpenAI Chat Model1` → `RAG` → `OpenAI Chat Model2` → `Retrive Agent`.  
   - Add Tool Vector Store node `RAG`:  
     - Name: `company_data`  
     - Description: "Retrieve data about company knowledge from vector store".  
   - Add Respond to Webhook node `Webhook RAG response`:  
     - Connect `Retrive Agent` → `Webhook RAG response`.  

5. **Qdrant Collection Setup and Document Ingestion**  
   - Add Manual Trigger node `When clicking ‘Test workflow’`.  
   - Add HTTP Request node `Create collection`:  
     - POST to `https://QDRANTURL/collections/COLLECTION` with empty filter.  
     - Authenticate with HTTP Header Auth (API key).  
     - Connect `When clicking ‘Test workflow’` → `Create collection`.  
   - Add HTTP Request node `Refresh collection`:  
     - POST to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter.  
     - Same auth as above.  
     - Connect `Create collection` → `Refresh collection`.  
   - Add Google Drive node `Get folder`:  
     - List files in folder `test-whatsapp` on "My Drive".  
     - Authenticate with Google Drive OAuth2.  
     - Connect `Refresh collection` → `Get folder`.  
   - Add Google Drive node `Download Files`:  
     - Download files by ID, convert Google Docs to plain text.  
     - Connect `Get folder` → `Download Files`.  
   - Add Text Splitter node `Token Splitter`:  
     - Chunk size: 300 tokens, overlap: 30 tokens.  
     - Connect `Download Files` → `Token Splitter`.  
   - Add Document Default Data Loader node `Default Data Loader`:  
     - Converts binary data to document format.  
     - Connect `Token Splitter` → `Default Data Loader`.  
   - Add Embeddings OpenAI node `Embeddings OpenAI`:  
     - Authenticate with OpenAI API.  
     - Connect `Default Data Loader` → `Embeddings OpenAI`.  
   - Add Vector Store Qdrant node `Qdrant Vector Store`:  
     - Authenticate with Qdrant API.  
     - Collection: same as above.  
     - Connect `Embeddings OpenAI` → `Qdrant Vector Store`.  

6. **Voiceflow Integration**  
   - Register on [Voiceflow](https://www.voiceflow.com/).  
   - Create three "Capture" blocks named `n8n_order`, `n8n_appointment`, and `n8n_rag`.  
   - Configure each capture to call the corresponding n8n webhook URL.  
   - Test the agent and retrieve your project ID.  
   - Choose Chat or Voice widget and copy the installation script.  
   - Optionally, import a Twilio number to enable phone agent capabilities.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Voiceflow is a no-code platform for building conversational assistants across chat, voice, and phone channels.                   | https://www.voiceflow.com/                                                                       |
| Setup requires Google Calendar & Drive OAuth credentials, Qdrant vector DB, and OpenAI API key.                                  | See workflow prerequisites section                                                              |
| Qdrant collection and document ingestion must be initialized before RAG queries work correctly.                                  | Qdrant API URL and collection name must be configured in HTTP Request nodes                      |
| The system prompt for the RAG agent is tailored for an electronics store assistant with detailed guidelines and example dialogs. | Embedded in `Retrive Agent` node parameters                                                     |
| For phone agent capabilities, import a Twilio number and link it to your Voiceflow agent.                                        | See Sticky Note images and instructions                                                          |
| Contact for consulting and support: [info@n3w.it](mailto:info@n3w.it), LinkedIn: https://www.linkedin.com/in/davideboizza/       |                                                                                                 |
| Workflow images illustrating architecture and Voiceflow setup are embedded in sticky notes for visual reference.                | See Sticky Notes in workflow                                                                     |

---

This document provides a comprehensive, structured reference for understanding, reproducing, and modifying the multi-channel conversational agent workflow integrating Voiceflow, Google Calendar, Qdrant, OpenAI, and an order tracking API. It anticipates common failure modes and highlights configuration points critical for successful deployment.