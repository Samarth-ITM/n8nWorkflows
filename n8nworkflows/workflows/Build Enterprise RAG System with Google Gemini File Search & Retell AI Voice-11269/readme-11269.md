Build Enterprise RAG System with Google Gemini File Search & Retell AI Voice

https://n8nworkflows.xyz/workflows/build-enterprise-rag-system-with-google-gemini-file-search---retell-ai-voice-11269


# Build Enterprise RAG System with Google Gemini File Search & Retell AI Voice

### 1. Workflow Overview

This workflow builds an enterprise-level Retrieval-Augmented Generation (RAG) system leveraging Google Gemini’s file search capabilities combined with AI voice retelling. It automates ingestion, indexing, and querying of documents stored in Google Drive, enabling interactive AI-powered chat responses enriched with document context.

The workflow is logically divided into the following blocks:

- **1.1 Initialization and File Store Setup**: Establishes the Gemini File Search store and Google Sheets as metadata repositories.
- **1.2 File Ingestion and Indexing**: Monitors Google Drive for new files, downloads them, and uploads content to Gemini File Search.
- **1.3 Query Handling and AI Processing**: Receives chat messages or webhook triggers, processes input with AI models using the stored document context, and responds accordingly.
- **1.4 Integration and Response Delivery**: Sends AI-generated responses back to the initiating webhook or chat interface.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and File Store Setup

- **Overview:**  
Sets up the Gemini File Search store and initializes the Google Sheets file store to maintain metadata about indexed files.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Create Gemini File Search Store (HTTP Request)  
  - Store File Store (Google Sheets)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution to initiate the setup.  
    - Configuration: Default manual trigger parameters.  
    - Inputs: None  
    - Outputs: Connects to "Create Gemini File Search Store"  
    - Potential failures: User forgetting to execute; no failures expected from the node itself.
  
  - **Create Gemini File Search Store**  
    - Type: HTTP Request  
    - Role: Sends a request to create or initialize the Gemini File Search store.  
    - Configuration: Uses Gemini API endpoint (details abstracted), likely with authentication credentials set in n8n.  
    - Inputs: Trigger from manual node.  
    - Outputs: Connects to "Store File Store"  
    - Possible issues: API authentication errors, network timeouts, or rate limiting from Gemini API.
  
  - **Store File Store**  
    - Type: Google Sheets  
    - Role: Stores metadata of the Gemini File Search store or files for later reference.  
    - Configuration: Google Sheets node connected to a specific spreadsheet configured to track file metadata.  
    - Inputs: From "Create Gemini File Search Store"  
    - Outputs: None (end of this chain)  
    - Edge cases: Google Sheets API quota exceeded, auth token expiration.

---

#### 1.2 File Ingestion and Indexing

- **Overview:**  
Monitors Google Drive for new files, retrieves file metadata, downloads the files, and uploads their content to the Gemini File Search for indexing.

- **Nodes Involved:**  
  - Google Drive - RAG Source (Google Drive Trigger)  
  - Get File Search Store (Google Sheets)  
  - Initial File Upload to Gemini Search (HTTP Request)  
  - Download file (Google Drive)  
  - Upload the Actual File (HTTP Request)

- **Node Details:**

  - **Google Drive - RAG Source**  
    - Type: Google Drive Trigger  
    - Role: Watches a Google Drive folder or account for new or updated files to process.  
    - Configuration: Configured with Google Drive OAuth2 credentials; triggers on file changes.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get File Search Store"  
    - Possible failures: Auth token expiry, Google Drive API errors, network issues.

  - **Get File Search Store**  
    - Type: Google Sheets  
    - Role: Retrieves existing file metadata or store information to determine if file is already indexed.  
    - Configuration: Reads from a Google Sheet maintaining file metadata.  
    - Inputs: From "Google Drive - RAG Source"  
    - Outputs: Connects to "Initial File Upload to Gemini Search"  
    - Edge cases: Missing or corrupted metadata, read permission errors.

  - **Initial File Upload to Gemini Search**  
    - Type: HTTP Request  
    - Role: Initializes upload process to Gemini File Search, possibly obtaining upload URLs or session tokens.  
    - Configuration: Calls Gemini API with file metadata, setting up for actual content upload.  
    - Inputs: From "Get File Search Store"  
    - Outputs: Connects to "Download file"  
    - Failure modes: API errors, invalid metadata, session token expiry.

  - **Download file**  
    - Type: Google Drive  
    - Role: Downloads the actual file content from Google Drive for upload.  
    - Configuration: Uses Google Drive node with file ID from previous steps.  
    - Inputs: From "Initial File Upload to Gemini Search"  
    - Outputs: Connects to "Upload the Actual File"  
    - Edge cases: File deleted or inaccessible, download timeout, large file size handling.

  - **Upload the Actual File**  
    - Type: HTTP Request  
    - Role: Uploads the downloaded file content to Gemini File Search for indexing.  
    - Configuration: Uses upload URL or token from initial upload node, sends file as multipart/form-data or base64 payload.  
    - Inputs: From "Download file"  
    - Outputs: None (end of ingestion chain)  
    - Potential failures: Upload failure, partial uploads, API rate limits.

---

#### 1.3 Query Handling and AI Processing

- **Overview:**  
Handles incoming chat messages or webhook requests, prepares input data, runs AI RAG agent combining Google Gemini Chat Model and Search Tool to generate context-aware responses.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - Webhook (Webhook Trigger)  
  - Edit Fields (Set)  
  - AI Powered RAG Agent (LangChain Agent)  
  - Google Gemini Chat Model (LangChain LM Chat)  
  - Search Tool (HTTP Request Tool)  

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Listens for incoming chat messages as inputs for AI processing.  
    - Configuration: Webhook-based chat trigger, linked to LangChain AI nodes.  
    - Inputs: None (trigger)  
    - Outputs: Connects to "Edit Fields" to prepare data.  
    - Possible failures: Webhook connectivity, malformed messages.

  - **Webhook**  
    - Type: Standard Webhook  
    - Role: Alternative input method for receiving queries or requests from external systems.  
    - Configuration: HTTP endpoint exposing webhook URL for external calls.  
    - Inputs: None (trigger)  
    - Outputs: Connects to "Edit Fields"  
    - Edge cases: Unauthorized requests, invalid payloads.

  - **Edit Fields**  
    - Type: Set  
    - Role: Prepares and formats input data for the AI agent, e.g., setting variables, cleaning inputs.  
    - Configuration: Uses expressions to map incoming data fields to AI agent input schema.  
    - Inputs: From "Webhook" or "When chat message received"  
    - Outputs: Connects to "AI Powered RAG Agent"  
    - Failure modes: Expression evaluation errors, missing fields.

  - **AI Powered RAG Agent**  
    - Type: LangChain Agent  
    - Role: Core AI agent that orchestrates language model and search integration to generate enriched responses.  
    - Configuration: Integrates with both "Google Gemini Chat Model" and "Search Tool" nodes as AI language model and search tool respectively.  
    - Inputs: From "Edit Fields" (main), "Google Gemini Chat Model" (ai_languageModel), "Search Tool" (ai_tool)  
    - Outputs: Connects to "Respond to Webhook"  
    - Edge cases: Model API limits, search tool errors, inconsistent context.

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat  
    - Role: Provides AI language model capabilities using Google Gemini’s chat API.  
    - Configuration: Connected with Google Gemini credentials, set for chat completion.  
    - Inputs: From "AI Powered RAG Agent" (ai_languageModel)  
    - Outputs: Back to "AI Powered RAG Agent"  
    - Failure modes: API authentication errors, response timeouts.

  - **Search Tool**  
    - Type: HTTP Request Tool  
    - Role: Performs document search queries against the Gemini File Search store to provide context to the AI agent.  
    - Configuration: Configured with Gemini Search API endpoint, query parameters derived from AI agent.  
    - Inputs: From "AI Powered RAG Agent" (ai_tool)  
    - Outputs: Back to "AI Powered RAG Agent"  
    - Edge cases: Search failures, no results found, malformed queries.

---

#### 1.4 Integration and Response Delivery

- **Overview:**  
Delivers the AI-generated response back to the original requester via the webhook response node.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends the final AI-generated output back to the external system that invoked the webhook or chat interface.  
    - Configuration: Responds with the JSON payload from the "AI Powered RAG Agent" node.  
    - Inputs: From "AI Powered RAG Agent"  
    - Outputs: None (endpoint of response path)  
    - Edge cases: Webhook timeout, client disconnect, malformed response data.

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                             | Input Node(s)                  | Output Node(s)                 | Sticky Note                      |
|-----------------------------|-------------------------------|---------------------------------------------|--------------------------------|--------------------------------|---------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual start for setup                      | None                           | Create Gemini File Search Store |                                 |
| Create Gemini File Search Store  | HTTP Request                 | Initialize Gemini File Search store         | When clicking ‘Execute workflow’ | Store File Store               |                                 |
| Store File Store             | Google Sheets                 | Store file metadata                          | Create Gemini File Search Store | None                          |                                 |
| Google Drive - RAG Source    | Google Drive Trigger          | Trigger on new/updated Google Drive files   | None                           | Get File Search Store          |                                 |
| Get File Search Store        | Google Sheets                 | Retrieve file metadata                        | Google Drive - RAG Source      | Initial File Upload to Gemini Search |                                 |
| Initial File Upload to Gemini Search | HTTP Request               | Initialize file upload to Gemini             | Get File Search Store          | Download file                 |                                 |
| Download file               | Google Drive                  | Download actual file content                  | Initial File Upload to Gemini Search | Upload the Actual File         |                                 |
| Upload the Actual File       | HTTP Request                 | Upload file content to Gemini File Search    | Download file                 | None                          |                                 |
| When chat message received   | LangChain Chat Trigger        | Receive chat messages for AI processing       | None                          | Edit Fields                   |                                 |
| Webhook                     | Webhook                      | Alternative input method for queries          | None                          | Edit Fields                   |                                 |
| Edit Fields                 | Set                          | Prepare input data for AI agent                | Webhook, When chat message received | AI Powered RAG Agent          |                                 |
| AI Powered RAG Agent         | LangChain Agent              | Orchestrate AI model and search tool          | Edit Fields, Google Gemini Chat Model (ai_languageModel), Search Tool (ai_tool) | Respond to Webhook            |                                 |
| Google Gemini Chat Model     | LangChain LM Chat            | Provide language model output via Gemini API | AI Powered RAG Agent (ai_languageModel) | AI Powered RAG Agent          |                                 |
| Search Tool                 | HTTP Request Tool             | Perform document search queries                | AI Powered RAG Agent (ai_tool)| AI Powered RAG Agent          |                                 |
| Respond to Webhook          | Respond to Webhook            | Send AI response back to caller                | AI Powered RAG Agent          | None                          |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Execute workflow’  
   - Type: Manual Trigger  
   - No special parameters needed.

2. **Create HTTP Request Node to Initialize Gemini Store**  
   - Name: Create Gemini File Search Store  
   - Type: HTTP Request  
   - Configure authentication with Gemini API credentials.  
   - Set method and URL as per Gemini API specification for store creation.  
   - Connect input from manual trigger.

3. **Create Google Sheets Node to Store Metadata**  
   - Name: Store File Store  
   - Type: Google Sheets  
   - Configure Google Sheets credentials and specify spreadsheet and sheet for file metadata storage.  
   - Connect input from Gemini store creation node.

4. **Create Google Drive Trigger Node**  
   - Name: Google Drive - RAG Source  
   - Type: Google Drive Trigger  
   - Configure OAuth2 credentials for Google Drive access.  
   - Set trigger to monitor desired folder or entire drive for new or updated files.

5. **Create Google Sheets Node to Retrieve Metadata**  
   - Name: Get File Search Store  
   - Type: Google Sheets  
   - Configure to read metadata spreadsheet.  
   - Connect input from Google Drive trigger node.

6. **Create HTTP Request Node to Initialize File Upload**  
   - Name: Initial File Upload to Gemini Search  
   - Type: HTTP Request  
   - Configure Gemini API endpoint for file upload initialization.  
   - Authentication via Gemini credentials.  
   - Connect input from metadata retrieval node.

7. **Create Google Drive Node to Download File**  
   - Name: Download file  
   - Type: Google Drive  
   - Configure with Google Drive credentials.  
   - Use file ID from previous node to download content.  
   - Connect input from initial file upload node.

8. **Create HTTP Request Node to Upload File Content**  
   - Name: Upload the Actual File  
   - Type: HTTP Request  
   - Use obtained upload URL or token from initialization node.  
   - Configure to send file content (binary or base64).  
   - Connect input from download file node.

9. **Create LangChain Chat Trigger Node**  
   - Name: When chat message received  
   - Type: LangChain Chat Trigger  
   - Configure webhook parameters for receiving chat messages.

10. **Create Webhook Node**  
    - Name: Webhook  
    - Type: Webhook  
    - Configure webhook for alternate input method.

11. **Create Set Node to Prepare Input Fields**  
    - Name: Edit Fields  
    - Type: Set  
    - Use expressions to format incoming data from "Webhook" or "When chat message received" nodes.  
    - Connect inputs from both trigger nodes.

12. **Create LangChain Agent Node**  
    - Name: AI Powered RAG Agent  
    - Type: LangChain Agent  
    - Configure agent to use two integrations:  
      - AI Language Model: Connect to "Google Gemini Chat Model" node  
      - AI Tool: Connect to "Search Tool" node  
    - Connect input from "Edit Fields".

13. **Create LangChain LM Chat Node**  
    - Name: Google Gemini Chat Model  
    - Type: LangChain LM Chat  
    - Configure with Google Gemini credentials and chat model parameters.  
    - Connect input from AI Powered RAG Agent (ai_languageModel).

14. **Create HTTP Request Tool Node**  
    - Name: Search Tool  
    - Type: HTTP Request Tool  
    - Configure Gemini File Search API endpoint for search queries.  
    - Connect input from AI Powered RAG Agent (ai_tool).

15. **Create Respond to Webhook Node**  
    - Name: Respond to Webhook  
    - Type: Respond to Webhook  
    - Connect input from AI Powered RAG Agent.  
    - Configure to send final response JSON back to the caller.

16. **Connect Nodes in Workflow**  
    - Manual Trigger → Create Gemini File Search Store → Store File Store  
    - Google Drive Trigger → Get File Search Store → Initial File Upload to Gemini Search → Download file → Upload the Actual File  
    - Webhook → Edit Fields → AI Powered RAG Agent → Respond to Webhook  
    - When chat message received → Edit Fields  
    - Google Gemini Chat Model and Search Tool connected as ai_languageModel and ai_tool inputs to AI Powered RAG Agent.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                      |
|-----------------------------------------------------------------------------------------------|------------------------------------|
| Workflow uses Google Gemini API for advanced AI chat and file search capabilities.            | Gemini API documentation recommended for setup. |
| Google Drive OAuth2 credentials must be configured with appropriate scopes for file access.   | Google Drive API docs.              |
| Google Sheets used for metadata tracking, ensure proper quota and API limits are respected.   | Google Sheets API documentation.   |
| AI agent relies on LangChain integrations; ensure compatible n8n LangChain nodes are installed.| n8n LangChain node documentation.  |
| Possible bottlenecks include API rate limits and large file uploads requiring chunking logic. | Monitor API usage and file sizes.  |
| No sticky notes contained textual information or comments in this version of the workflow.    |                                 |

---

**Disclaimer:** The provided information is extracted solely from an automated n8n workflow and adheres to all applicable content policies. It contains no illegal or protected content. All data processed is lawful and public.