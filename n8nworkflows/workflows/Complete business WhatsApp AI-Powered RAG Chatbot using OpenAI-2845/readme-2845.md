Complete business WhatsApp AI-Powered RAG Chatbot using OpenAI

https://n8nworkflows.xyz/workflows/complete-business-whatsapp-ai-powered-rag-chatbot-using-openai-2845


# Complete business WhatsApp AI-Powered RAG Chatbot using OpenAI

### 1. Workflow Overview

This workflow implements a Business WhatsApp AI-powered Retrieval-Augmented Generation (RAG) chatbot using OpenAI and Qdrant vector database. It is designed to handle customer inquiries for an electronics store via WhatsApp, providing accurate product information, technical support, and customer service by leveraging a knowledge base of documents stored in Google Drive and indexed in Qdrant.

The workflow is logically divided into the following blocks:

- **1.1 Qdrant Collection Setup and Document Vectorization**: Initializes and refreshes the Qdrant collection, downloads documents from Google Drive, generates embeddings using OpenAI, and stores vectors in Qdrant.

- **1.2 Webhook Setup and Message Reception**: Configures webhooks to verify Meta’s callback URL and receive incoming WhatsApp messages.

- **1.3 Message Validation and Routing**: Checks if the incoming webhook payload contains a user message and routes accordingly.

- **1.4 AI Agent Processing and Response Generation**: Uses a conversational AI agent with a system prompt tailored for an electronics store, augmented by the knowledge base retrieved from Qdrant, to generate responses.

- **1.5 WhatsApp Response Sending**: Sends the AI-generated or fallback messages back to the user via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Qdrant Collection Setup and Document Vectorization

**Overview:**  
This block initializes the Qdrant vector database collection, clears existing data, downloads documents from Google Drive, generates embeddings using OpenAI, and inserts the vectors into Qdrant. It prepares the knowledge base for retrieval during conversations.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Create collection  
- Refresh collection  
- Get folder  
- Download Files  
- Embeddings OpenAI  
- Default Data Loader  
- Token Splitter  
- Qdrant Vector Store  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing and initialization  
  - Configuration: No parameters  
  - Inputs: None  
  - Outputs: Triggers "Create collection" and "Refresh collection" nodes  
  - Edge Cases: None  

- **Create collection**  
  - Type: HTTP Request  
  - Role: Creates a new collection in Qdrant for storing vectors  
  - Configuration: POST request to `https://QDRANTURL/collections/COLLECTION` with empty filter JSON body  
  - Credentials: HTTP Header Auth with Qdrant API key  
  - Inputs: Triggered by manual node  
  - Outputs: None (proceeds to Refresh collection)  
  - Edge Cases: API errors, authentication failures, invalid URL or collection name  

- **Refresh collection**  
  - Type: HTTP Request  
  - Role: Deletes all points in the Qdrant collection to refresh data  
  - Configuration: POST request to `https://QDRANTURL/collections/COLLECTION/points/delete` with empty filter JSON body  
  - Credentials: Same as Create collection  
  - Inputs: Triggered by manual node  
  - Outputs: Triggers "Get folder" node  
  - Edge Cases: API errors, authentication failures  

- **Get folder**  
  - Type: Google Drive  
  - Role: Retrieves files from a specified Google Drive folder (e.g., "test-whatsapp")  
  - Configuration: Filters by Drive ID "My Drive" and folder ID "test-whatsapp"  
  - Credentials: Google Drive OAuth2  
  - Inputs: Triggered by Refresh collection  
  - Outputs: Passes file metadata to "Download Files"  
  - Edge Cases: Folder not found, permission errors, API rate limits  

- **Download Files**  
  - Type: Google Drive  
  - Role: Downloads each file from the folder as plain text  
  - Configuration: Converts Google Docs to "text/plain" format  
  - Credentials: Google Drive OAuth2  
  - Inputs: Receives file ID from "Get folder"  
  - Outputs: Passes binary file data to "Qdrant Vector Store" via embeddings chain  
  - Edge Cases: Download failures, conversion errors, permission issues  

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings for document content  
  - Configuration: Uses OpenAI API with default embedding model  
  - Credentials: OpenAI API key  
  - Inputs: Receives document text/binary from "Download Files"  
  - Outputs: Passes embeddings to "Qdrant Vector Store"  
  - Edge Cases: API rate limits, invalid input data, network errors  

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Loads document data for vectorization pipeline  
  - Configuration: Accepts binary data  
  - Inputs: Connected downstream of "Token Splitter" (see below)  
  - Outputs: Passes loaded documents to "Qdrant Vector Store"  
  - Edge Cases: Data format errors  

- **Token Splitter**  
  - Type: Text Splitter (Token-based)  
  - Role: Splits documents into chunks of 300 tokens with 30 tokens overlap for better embedding granularity  
  - Configuration: chunkSize=300, chunkOverlap=30  
  - Inputs: Receives text from "Default Data Loader"  
  - Outputs: Passes chunks to "Embeddings OpenAI"  
  - Edge Cases: Improper splitting if input is malformed  

- **Qdrant Vector Store**  
  - Type: Vector Store (Qdrant)  
  - Role: Inserts vector embeddings into the Qdrant collection  
  - Configuration: Mode set to "insert", collection name parameterized  
  - Credentials: Qdrant API key  
  - Inputs: Receives embeddings and document metadata  
  - Outputs: None (end of vectorization chain)  
  - Edge Cases: API errors, insertion failures, authentication issues  

---

#### 2.2 Webhook Setup and Message Reception

**Overview:**  
This block sets up two webhooks: one for verification of Meta’s callback URL via GET requests, and another for receiving WhatsApp messages via POST requests.

**Nodes Involved:**  
- Verify  
- Respond  
- Respond to Webhook  

**Node Details:**

- **Verify**  
  - Type: Webhook  
  - Role: Receives GET requests from Meta to verify the webhook URL  
  - Configuration: Path set to a unique webhook ID, response mode set to "responseNode"  
  - Inputs: External GET requests  
  - Outputs: Triggers "Respond to Webhook" node  
  - Edge Cases: Incorrect verification response, URL mismatch  

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends back the verification challenge token to Meta  
  - Configuration: Responds with the value of `hub.challenge` from query parameters as plain text  
  - Inputs: Triggered by "Verify" node  
  - Outputs: None (ends verification flow)  
  - Edge Cases: Missing or malformed query parameters  

- **Respond**  
  - Type: Webhook  
  - Role: Receives POST requests from Meta containing WhatsApp messages and status updates  
  - Configuration: Same webhook path as "Verify", HTTP method POST  
  - Inputs: External POST requests  
  - Outputs: Triggers "is Message?" node  
  - Edge Cases: Payload format changes, missing message data  

---

#### 2.3 Message Validation and Routing

**Overview:**  
This block checks if the incoming webhook payload contains a user message. If yes, it proceeds to AI processing; if not, it sends a fallback message.

**Nodes Involved:**  
- is Message?  
- Only message  

**Node Details:**

- **is Message?**  
  - Type: If  
  - Role: Checks if the JSON path `body.entry[0].changes[0].value.messages[0]` exists, indicating a user message  
  - Configuration: Condition uses "exists" operator on the specified JSON path  
  - Inputs: Triggered by "Respond" webhook  
  - Outputs:  
    - True branch: triggers "AI Agent" node  
    - False branch: triggers "Only message" node  
  - Edge Cases: Payload structure changes, missing fields causing false negatives  

- **Only message**  
  - Type: WhatsApp  
  - Role: Sends a fallback text message "You can only send text messages" to the user if no valid message is detected  
  - Configuration: Sends text message to phone number extracted from webhook payload  
  - Credentials: WhatsApp API credentials  
  - Inputs: Triggered by "is Message?" false branch  
  - Outputs: None  
  - Edge Cases: Invalid phone number, WhatsApp API errors  

---

#### 2.4 AI Agent Processing and Response Generation

**Overview:**  
This block processes the user message with a conversational AI agent configured with a detailed system prompt tailored for an electronics store. It uses a RAG approach by retrieving relevant documents from Qdrant to augment the AI’s responses.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- Embeddings OpenAI2  
- Retrive Qdrant Vector Store  
- OpenAI Chat Model1  
- RAG  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Main conversational AI node that receives user text, uses system prompt, memory, and tools to generate responses  
  - Configuration:  
    - Text input extracted from incoming WhatsApp message text  
    - System message defines assistant role, guidelines for product info, support, customer service, knowledge base usage, tone, and limitations  
    - Uses conversational agent type with memory buffer window  
  - Inputs: Triggered by "is Message?" true branch  
  - Outputs: Triggers "Send" node to send response  
  - Edge Cases: Expression evaluation errors, missing input text, API failures  

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Completion  
  - Role: Provides language model capabilities (gpt-4o-mini) for the AI Agent  
  - Configuration: Model set to "gpt-4o-mini"  
  - Credentials: OpenAI API key  
  - Inputs: Connected as language model for AI Agent  
  - Outputs: AI Agent receives generated completions  
  - Edge Cases: API rate limits, model unavailability  

- **Window Buffer Memory**  
  - Type: Memory Buffer Window  
  - Role: Maintains conversational context window for AI Agent to enable multi-turn dialogue  
  - Configuration: Default settings  
  - Inputs: Connected as memory for AI Agent  
  - Outputs: Provides context to AI Agent  
  - Edge Cases: Memory overflow or truncation issues  

- **Embeddings OpenAI2**  
  - Type: OpenAI Embeddings  
  - Role: Generates embeddings for query text to retrieve relevant documents from Qdrant  
  - Configuration: Default OpenAI embedding model  
  - Credentials: OpenAI API key  
  - Inputs: Receives query text from AI Agent or RAG tool  
  - Outputs: Passes embeddings to "Retrive Qdrant Vector Store"  
  - Edge Cases: API errors, invalid input  

- **Retrive Qdrant Vector Store**  
  - Type: Vector Store (Qdrant)  
  - Role: Retrieves relevant documents from Qdrant based on query embeddings  
  - Configuration: Uses collection named "COLLECTION"  
  - Credentials: Qdrant API key  
  - Inputs: Receives embeddings from "Embeddings OpenAI2"  
  - Outputs: Passes retrieved documents to "RAG" tool  
  - Edge Cases: Retrieval failures, API errors  

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Completion  
  - Role: Provides language model capabilities for the RAG tool  
  - Configuration: Model "gpt-4o-mini"  
  - Credentials: OpenAI API key  
  - Inputs: Connected as language model for "RAG" tool  
  - Outputs: Provides completions to "RAG" tool  
  - Edge Cases: API limits, model errors  

- **RAG**  
  - Type: Vector Store Tool  
  - Role: Combines retrieved documents with language model to generate augmented responses  
  - Configuration: Named "company_data" with description "Retrieve data about company knowledge from vector store"  
  - Inputs: Receives documents from "Retrive Qdrant Vector Store" and language model from "OpenAI Chat Model1"  
  - Outputs: Provides augmented response text to "AI Agent" as a tool output  
  - Edge Cases: Tool execution errors, missing documents  

---

#### 2.5 WhatsApp Response Sending

**Overview:**  
This block sends the AI-generated response back to the user via WhatsApp API.

**Nodes Involved:**  
- Send  

**Node Details:**

- **Send**  
  - Type: WhatsApp  
  - Role: Sends text message response to the user’s WhatsApp number  
  - Configuration:  
    - Text body set dynamically from AI Agent output (`$json.output`)  
    - Recipient phone number extracted from webhook payload  
  - Credentials: WhatsApp API credentials  
  - Inputs: Triggered by "AI Agent" output  
  - Outputs: None  
  - Edge Cases: Invalid phone number, WhatsApp API errors, message formatting issues  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                  | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                              |
|-------------------------|----------------------------------|-------------------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start for collection setup and refresh   |                             | Create collection, Refresh collection |                                                                                                        |
| Create collection        | HTTP Request                     | Creates Qdrant collection                        | When clicking ‘Test workflow’ | Refresh collection          | # STEP 1: Create Qdrant Collection. Change QDRANTURL and COLLECTION                                     |
| Refresh collection       | HTTP Request                     | Deletes all points in Qdrant collection          | When clicking ‘Test workflow’ | Get folder                 |                                                                                                        |
| Get folder               | Google Drive                     | Retrieves files from Google Drive folder          | Refresh collection           | Download Files             | # STEP 2: Documents vectorization with Qdrant and Google Drive. Change QDRANTURL and COLLECTION        |
| Download Files           | Google Drive                     | Downloads files as plain text                      | Get folder                   | Qdrant Vector Store        |                                                                                                        |
| Embeddings OpenAI        | OpenAI Embeddings                | Generates embeddings for documents                | Download Files               | Qdrant Vector Store        |                                                                                                        |
| Default Data Loader      | Document Data Loader             | Loads document data for vectorization             | Token Splitter               | Qdrant Vector Store        |                                                                                                        |
| Token Splitter           | Text Splitter                   | Splits text into chunks for embedding             | Default Data Loader          | Embeddings OpenAI          |                                                                                                        |
| Qdrant Vector Store      | Vector Store (Qdrant)            | Inserts embeddings into Qdrant collection         | Embeddings OpenAI, Default Data Loader |                            |                                                                                                        |
| Verify                  | Webhook                         | Verifies Meta webhook URL via GET                  |                             | Respond to Webhook         | # STEP 3: Create Webhook. See https://developers.facebook.com/apps/                                    |
| Respond to Webhook       | Respond to Webhook               | Sends back verification challenge token           | Verify                      |                            |                                                                                                        |
| Respond                  | Webhook                         | Receives WhatsApp messages via POST                |                             | is Message?                | # STEP 4: Respond webhook receives POST requests from Meta regarding WhatsApp messages                 |
| is Message?              | If                              | Checks if incoming payload contains a user message | Respond                     | AI Agent (true), Only message (false) |                                                                                                        |
| Only message             | WhatsApp                        | Sends fallback message if no valid user message   | is Message? (false)          |                            |                                                                                                        |
| AI Agent                 | LangChain Agent                 | Processes user message with conversational AI     | is Message? (true)           | Send                       | # Configure AI Agent: Set system prompt and chat model. Optionally set tools                            |
| OpenAI Chat Model        | OpenAI Chat Completion          | Provides language model for AI Agent               | AI Agent                    | AI Agent                   |                                                                                                        |
| Window Buffer Memory     | Memory Buffer Window            | Maintains conversation context                      |                             | AI Agent                   |                                                                                                        |
| Embeddings OpenAI2       | OpenAI Embeddings               | Embeds query text for document retrieval           |                             | Retrive Qdrant Vector Store |                                                                                                        |
| Retrive Qdrant Vector Store | Vector Store (Qdrant)            | Retrieves relevant documents from Qdrant           | Embeddings OpenAI2           | RAG                        |                                                                                                        |
| OpenAI Chat Model1       | OpenAI Chat Completion          | Language model for RAG tool                         |                            | RAG                        |                                                                                                        |
| RAG                     | Vector Store Tool               | Combines retrieved docs and LM to generate answers | Retrive Qdrant Vector Store, OpenAI Chat Model1 | AI Agent (ai_tool)          |                                                                                                        |
| Send                    | WhatsApp                        | Sends AI-generated response to user                 | AI Agent                    |                            |                                                                                                        |
| Sticky Note              | Sticky Note                    | Instructional notes on webhook setup and message handling |                             |                            | # STEP 4: Respond webhook receives POST requests from Meta regarding WhatsApp messages                 |
| Sticky Note1             | Sticky Note                    | Instructions for webhook creation                   |                             |                            | # STEP 3: Create Webhook. See https://developers.facebook.com/apps/                                    |
| Sticky Note2             | Sticky Note                    | Webhook configuration reminders                     |                             |                            | Important! Configure Verify webhook as GET and Respond webhook as POST with same URL                   |
| Sticky Note3             | Sticky Note                    | Reminder to change QDRANTURL and COLLECTION         |                             |                            | # STEP 1: Create Qdrant Collection                                                                    |
| Sticky Note4             | Sticky Note                    | Reminder to change QDRANTURL and COLLECTION for vectorization |                             |                            | # STEP 2: Documents vectorization with Qdrant and Google Drive                                        |
| Sticky Note5             | Sticky Note                    | AI Agent configuration reminder                      |                             |                            | Configure AI Agent: Set system prompt and chat model. Optionally set tools                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the collection setup and refresh process  

2. **Create HTTP Request Node "Create collection"**  
   - Method: POST  
   - URL: `https://QDRANTURL/collections/COLLECTION` (replace placeholders)  
   - Body: JSON `{ "filter": {} }`  
   - Headers: `Content-Type: application/json`  
   - Authentication: HTTP Header Auth with Qdrant API key credentials  
   - Connect input from Manual Trigger  

3. **Create HTTP Request Node "Refresh collection"**  
   - Method: POST  
   - URL: `https://QDRANTURL/collections/COLLECTION/points/delete`  
   - Body: JSON `{ "filter": {} }`  
   - Headers: `Content-Type: application/json`  
   - Authentication: Same as above  
   - Connect input from Manual Trigger  

4. **Create Google Drive Node "Get folder"**  
   - Resource: File/Folder  
   - Operation: List files in folder  
   - Filter: Drive ID "My Drive", Folder ID "test-whatsapp" (replace as needed)  
   - Credentials: Google Drive OAuth2  
   - Connect input from "Refresh collection"  

5. **Create Google Drive Node "Download Files"**  
   - Operation: Download  
   - File ID: Use expression `{{$json["id"]}}` from "Get folder" output  
   - Enable Google Docs conversion to "text/plain"  
   - Credentials: Google Drive OAuth2  
   - Connect input from "Get folder"  

6. **Create LangChain Node "Token Splitter"**  
   - Type: Text Splitter (Token-based)  
   - Chunk Size: 300 tokens  
   - Chunk Overlap: 30 tokens  

7. **Create LangChain Node "Default Data Loader"**  
   - Data Type: Binary  
   - Connect input from "Token Splitter"  

8. **Create LangChain Node "Embeddings OpenAI"**  
   - Use default OpenAI embedding model  
   - Credentials: OpenAI API key  
   - Connect input from "Default Data Loader"  

9. **Create LangChain Node "Qdrant Vector Store"**  
   - Mode: Insert  
   - Collection: Use your collection name  
   - Credentials: Qdrant API key  
   - Connect input from "Embeddings OpenAI"  

10. **Create Webhook Node "Verify"**  
    - HTTP Method: GET  
    - Path: Unique webhook ID (e.g., `f0d2e6f6-8fda-424d-b377-0bd191343c20`)  
    - Response Mode: Response Node  

11. **Create Respond to Webhook Node "Respond to Webhook"**  
    - Respond with: Text  
    - Response Body: Expression `={{ $json.query['hub.challenge'] }}`  
    - Connect input from "Verify"  

12. **Create Webhook Node "Respond"**  
    - HTTP Method: POST  
    - Path: Same as "Verify" webhook path  
    - Connect input from external POST requests  

13. **Create If Node "is Message?"**  
    - Condition: Check if `body.entry[0].changes[0].value.messages[0]` exists  
    - Connect input from "Respond"  

14. **Create WhatsApp Node "Only message"**  
    - Operation: Send text message  
    - Text Body: "You can only send text messages"  
    - Recipient Phone Number: Expression `={{ $('Respond').item.json.body.entry[0].changes[0].value.contacts[0].wa_id }}`  
    - Credentials: WhatsApp API  
    - Connect input from "is Message?" false branch  

15. **Create LangChain Agent Node "AI Agent"**  
    - Text Input: Expression `={{ $('Respond').item.json.body.entry[0].changes[0].value.messages[0].text.body }}`  
    - Agent Type: Conversational Agent  
    - System Message: Use detailed electronics store assistant prompt (as provided)  
    - Prompt Type: Define  
    - Connect input from "is Message?" true branch  

16. **Create OpenAI Chat Model Node "OpenAI Chat Model"**  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API  
    - Connect as language model for "AI Agent"  

17. **Create Window Buffer Memory Node**  
    - Default settings  
    - Connect as memory for "AI Agent"  

18. **Create OpenAI Embeddings Node "Embeddings OpenAI2"**  
    - Credentials: OpenAI API  
    - Connect input from "AI Agent" or query text for retrieval  

19. **Create Vector Store Node "Retrive Qdrant Vector Store"**  
    - Collection: Your Qdrant collection  
    - Credentials: Qdrant API  
    - Connect input from "Embeddings OpenAI2"  

20. **Create OpenAI Chat Model Node "OpenAI Chat Model1"**  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API  
    - Connect as language model for "RAG" tool  

21. **Create Vector Store Tool Node "RAG"**  
    - Name: company_data  
    - Description: Retrieve data about company knowledge from vector store  
    - Connect inputs from "Retrive Qdrant Vector Store" and "OpenAI Chat Model1"  
    - Connect output to "AI Agent" as ai_tool  

22. **Create WhatsApp Node "Send"**  
    - Operation: Send text message  
    - Text Body: Expression `={{ $json.output }}` (AI Agent output)  
    - Recipient Phone Number: Expression `={{ $('Respond').item.json.body.entry[0].changes[0].value.contacts[0].wa_id }}`  
    - Credentials: WhatsApp API  
    - Connect input from "AI Agent" output  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Configure webhook nodes so that both *Verify* and *Respond* use the same URL; *Verify* uses GET, *Respond* uses POST | Sticky Note2 in workflow                                                                         |
| Create webhook in Meta for Developers App, add production webhook URL, and verify with GET request  | Sticky Note1; https://developers.facebook.com/apps/                                             |
| Change `QDRANTURL` and `COLLECTION` variables before running collection setup and vectorization     | Sticky Note3 and Sticky Note4                                                                    |
| AI Agent system prompt tailored for electronics store with guidelines for product info, support, and customer service | Sticky Note5                                                                                     |
| The workflow uses OpenAI model "gpt-4o-mini" for chat completions and embeddings                     | Credential setup required for OpenAI API                                                        |
| WhatsApp API credentials must be configured for sending and receiving messages                       | Credential setup required for WhatsApp API                                                      |

---

This documentation provides a detailed, stepwise understanding of the Business WhatsApp AI RAG Chatbot workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.