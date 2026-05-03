RAG-powered Document Chat with Google Drive, OpenAI, and Pinecone Assistant

https://n8nworkflows.xyz/workflows/rag-powered-document-chat-with-google-drive--openai--and-pinecone-assistant-9942


# RAG-powered Document Chat with Google Drive, OpenAI, and Pinecone Assistant

---

### 1. Workflow Overview

This workflow enables a Retrieval-Augmented Generation (RAG) powered chat assistant that integrates Google Drive, OpenAI GPT-4, and Pinecone vector database services to allow users to interact conversationally with their own documents stored on Google Drive. When files are added or updated in a specified Google Drive folder, the workflow ingests those files into a Pinecone Assistant for vector-based semantic search, enabling context-aware responses to user queries via a chat interface.

The workflow consists of two main logical blocks:

- **1.1 Document Ingestion and Synchronization:** Watches a specific Google Drive folder for newly added or updated files, downloads them, uploads them to the Pinecone Assistant, and manages file synchronization by deleting outdated versions in Pinecone.

- **1.2 Chat Query Handling with AI Agent:** Listens for chat inputs, routes queries to an AI Agent that uses OpenAI’s GPT-4 chat model combined with the Pinecone Assistant for context retrieval, and maintains conversation memory to provide coherent, context-aware answers.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Synchronization

**Overview:**  
This block monitors a designated Google Drive folder (`n8n-pinecone-demo`) for file additions or updates. Upon detection, it downloads the file, uploads it to the Pinecone Assistant (which manages vector embeddings), checks the processing status, deletes old versions if needed, and ensures synchronization between Google Drive and Pinecone.

**Nodes Involved:**  
- File added (Google Drive Trigger)  
- File updated (Google Drive Trigger)  
- Download file (Google Drive)  
- Upload file to assistant (HTTP Request to Pinecone Assistant)  
- Get file to delete (HTTP Request to Pinecone Assistant)  
- Delete file from assistant (HTTP Request to Pinecone Assistant)  
- Set file data (Set)  
- Check file status (HTTP Request to Pinecone Assistant)  
- End if status Available (Switch)  
- Wait (Wait)  
- Sticky Note1 (Informational)

**Node Details:**

- **File added**  
  - Type: Google Drive Trigger  
  - Role: Triggers workflow when a new file is created in the specified folder.  
  - Config: Watches folder `n8n-pinecone-demo`, polls every minute.  
  - Inputs: None (trigger)  
  - Outputs: File metadata JSON  
  - Failures: OAuth errors, permission issues, polling delays.

- **File updated**  
  - Type: Google Drive Trigger  
  - Role: Triggers workflow when a file is updated in the specified folder.  
  - Config: Watches folder `n8n-pinecone-demo`, polls every minute.  
  - Inputs: None (trigger)  
  - Outputs: Updated file metadata JSON  
  - Failures: Same as File added.

- **Download file**  
  - Type: Google Drive Node  
  - Role: Downloads the file from Google Drive using file ID from trigger nodes.  
  - Config: Uses file ID and file name from trigger JSON.  
  - Inputs: File metadata (from triggers)  
  - Outputs: Binary file data for upload  
  - Failures: File not found, access denied, network timeout.

- **Upload file to assistant**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded file to Pinecone Assistant for indexing.  
  - Config: POST to Pinecone Assistant files endpoint, sends file as multipart form-data, includes metadata with external_file_id and external_file_name derived from Google Drive file properties. Uses predefined Pinecone API credentials.  
  - Inputs: Binary data from Download file, file metadata from triggers  
  - Outputs: Response JSON with file upload status and ID  
  - Failures: Authentication errors, upload timeout, API rate limits.

- **Get file to delete**  
  - Type: HTTP Request  
  - Role: Queries Pinecone Assistant for any existing files with the same external_file_id to identify old versions to delete.  
  - Config: GET with filter query parameter matching external_file_id from updated file metadata. Uses Pinecone API credentials.  
  - Inputs: Updated file metadata  
  - Outputs: JSON list of matching files in Pinecone Assistant  
  - Failures: API errors, malformed query.

- **Delete file from assistant**  
  - Type: HTTP Request  
  - Role: Deletes the old file version from Pinecone Assistant using file ID obtained from Get file to delete.  
  - Config: DELETE request to Pinecone Assistant files endpoint with file ID path parameter. Uses Pinecone API credentials.  
  - Inputs: File ID from Get file to delete response  
  - Outputs: Text confirmation of deletion  
  - Failures: Authorization errors, file not found, race conditions.

- **Set file data**  
  - Type: Set  
  - Role: Prepares JSON object with file ID and name from File updated trigger, standardizes data for downstream nodes.  
  - Config: Expression sets `id` and `name` fields.  
  - Inputs: File updated trigger JSON  
  - Outputs: JSON with file data  
  - Failures: Expression evaluation errors.

- **Check file status**  
  - Type: HTTP Request  
  - Role: Polls the Pinecone Assistant for the status of the uploaded file (e.g., Processing, Available).  
  - Config: GET request to Pinecone Assistant file status endpoint using file ID, with Pinecone credentials.  
  - Inputs: File ID from Upload file to assistant node  
  - Outputs: JSON with status field  
  - Failures: Network or API errors, invalid file ID.

- **End if status Available**  
  - Type: Switch  
  - Role: Routes workflow based on file status: if `Available`, ends polling; if `Processing`, triggers Wait node to retry later.  
  - Config: Checks if `$json.status` equals `"Available"` or `"Processing"`.  
  - Inputs: Status JSON from Check file status  
  - Outputs: Two branches: Available (ends), Processing (loops to Wait)  
  - Failures: Unexpected status values.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow for 3 seconds before rechecking file status.  
  - Config: Fixed 3-second wait time.  
  - Inputs: Triggered when file status is Processing  
  - Outputs: Single output to Check file status node for polling loop  
  - Failures: None typical.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Provides a high-level descriptive comment for this ingestion block: "Add new or update files from Google Drive to Pinecone Assistant."  
  - Inputs/Outputs: None  

#### 2.2 Chat Query Handling with AI Agent

**Overview:**  
This block enables interactive chat with the user’s documents by listening for chat requests, invoking an AI Agent that combines OpenAI GPT-4 chat capabilities with the Pinecone Assistant for contextual document search, and maintains conversation memory to deliver coherent multi-turn dialogues.

**Nodes Involved:**  
- Chat input (LangChain Chat Trigger)  
- AI Agent (LangChain Agent)  
- OpenAI Chat Model (LangChain LM Chat OpenAI)  
- Pinecone Assistant (LangChain MCP Client Tool)  
- Conversation Memory (LangChain Memory Buffer Window)  
- Sticky Note2 (Informational)  
- Sticky Note7 (Setup guide and instructions)

**Node Details:**

- **Chat input**  
  - Type: LangChain Chat Trigger  
  - Role: Webhook node that receives user chat messages, starting the conversational AI process.  
  - Config: Uses a webhook with ID `1bdcb9d4-a367-4d25-9405-ab765e595bf4` to receive input. No special options configured.  
  - Inputs: External chat messages via webhook  
  - Outputs: User message JSON to AI Agent  
  - Failures: Webhook connectivity, payload format errors.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Central orchestrator node that integrates language model, vector search tool, and memory to generate responses.  
  - Config: Default options; internally references connected AI language model, tool, and memory nodes via node connections.  
  - Inputs: Chat input messages, plus ai_languageModel, ai_tool, ai_memory inputs from respective nodes  
  - Outputs: AI-generated chat responses  
  - Failures: API quota limits, timeouts, misconfigurations of sub-components.

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides GPT-4 based chat completions to the AI Agent.  
  - Config: Uses model `gpt-4.1-mini`, linked via OpenAI API credential. No additional options set.  
  - Inputs: Prompts from AI Agent  
  - Outputs: Generated text completions to AI Agent  
  - Failures: API key invalid, rate limits, model deprecation.

- **Pinecone Assistant**  
  - Type: LangChain MCP Client Tool  
  - Role: Acts as vector search tool connected to Pinecone Assistant endpoint for context retrieval.  
  - Config: Connects to Pinecone Assistant endpoint URL `https://prod-1-data.ke.pinecone.io/mcp/assistants/n8n-assistant` with bearer token authentication. Uses HTTP streaming transport.  
  - Inputs: Queries from AI Agent  
  - Outputs: Retrieved context for AI Agent  
  - Failures: Authentication issues, endpoint unavailability.

- **Conversation Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains recent conversation history to support multi-turn dialogue coherence.  
  - Config: Default buffer window (no parameters explicitly set).  
  - Inputs: Conversation messages from AI Agent  
  - Outputs: Memory context to AI Agent  
  - Failures: Memory overload, excessive token usage.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Describes this block simply as "Chat with your docs."  
  - Inputs/Outputs: None  

- **Sticky Note7**  
  - Type: Sticky Note  
  - Role: Provides comprehensive setup instructions, prerequisites, and customization tips for this workflow.  
  - Content includes:  
    - Overview of functionality  
    - Prerequisites: Pinecone, Google Drive API, OpenAI accounts and keys  
    - Setup steps for Pinecone Assistant, OAuth2 credentials, API keys  
    - Folder naming requirement (`n8n-pinecone-demo`)  
    - Activation and testing instructions  
    - Customization ideas  
    - Support links to Pinecone community and GitHub issues  

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                           | Input Node(s)            | Output Node(s)             | Sticky Note                               |
|-------------------------|----------------------------------|-----------------------------------------|--------------------------|----------------------------|-------------------------------------------|
| File added              | Google Drive Trigger             | Trigger on new file in Drive folder     | -                        | Download file              |                                           |
| File updated            | Google Drive Trigger             | Trigger on file update in Drive folder  | -                        | Get file to delete         |                                           |
| Download file           | Google Drive                    | Download file binary from Drive         | File added, Set file data | Upload file to assistant   |                                           |
| Upload file to assistant | HTTP Request                    | Upload file to Pinecone Assistant        | Download file            | Check file status          |                                           |
| Get file to delete      | HTTP Request                    | Query Pinecone files to find old versions| File updated             | Delete file from assistant |                                           |
| Delete file from assistant| HTTP Request                   | Delete old file version from Pinecone    | Get file to delete       | Set file data              |                                           |
| Set file data           | Set                            | Prepare file ID and name JSON            | Delete file from assistant| Download file              |                                           |
| Check file status       | HTTP Request                    | Poll file processing status in Pinecone  | Upload file to assistant | End if status Available    |                                           |
| End if status Available | Switch                         | Branch on file status (Available / Processing) | Check file status     | Wait (Processing branch)   |                                           |
| Wait                    | Wait                           | Delay before polling again                | End if status Available  | Check file status          |                                           |
| Chat input              | LangChain Chat Trigger          | Receive chat input from user              | -                        | AI Agent                  |                                           |
| AI Agent                | LangChain Agent                | Core conversational AI orchestrator      | Chat input, LM/Tool/Memory| -                         |                                           |
| OpenAI Chat Model       | LangChain LM Chat OpenAI        | Generate GPT-4 based chat completions    | AI Agent (ai_languageModel)| AI Agent                 |                                           |
| Pinecone Assistant      | LangChain MCP Client Tool        | Vector search retrieval via Pinecone     | AI Agent (ai_tool)       | AI Agent                   |                                           |
| Conversation Memory     | LangChain Memory Buffer Window  | Maintain conversation context            | AI Agent (ai_memory)     | AI Agent                   |                                           |
| Sticky Note1            | Sticky Note                    | Describes ingestion block                 | -                        | -                         | "1. Add new or update files from Google Drive to Pinecone Assistant" |
| Sticky Note2            | Sticky Note                    | Describes chat block                      | -                        | -                         | "2. Chat with your docs"                   |
| Sticky Note7            | Sticky Note                    | Setup instructions and project overview | -                        | -                         | Detailed notes with setup steps and links |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers:**
   - Add two Google Drive Trigger nodes named `File added` and `File updated`.
   - Configure both to watch the folder named `n8n-pinecone-demo` in your Google Drive root.
   - Set the `File added` trigger for event `fileCreated`.
   - Set the `File updated` trigger for event `fileUpdated`.
   - Set polling interval to every minute.
   - Assign Google Drive OAuth2 credentials (named `Google Drive account`).

2. **Download File Node:**
   - Add a Google Drive node named `Download file`.
   - Set operation to `download`.
   - For the file ID, use expression referencing trigger JSON: `={{ $json.id }}`.
   - For file name, use expression: `={{ $json.name }}`.
   - Connect outputs of `File added` and `Set file data` nodes to `Download file`.

3. **Upload File to Pinecone Assistant:**
   - Add an HTTP Request node named `Upload file to assistant`.
   - Set method to POST.
   - Set URL to: `https://prod-1-data.ke.pinecone.io/assistant/files/n8n-assistant`.
   - Set Content-Type to `multipart/form-data`.
   - In body parameters, add a parameter named `file` of type `formBinaryData` linked to binary field `data` from `Download file`.
   - Add query parameter `metadata` with JSON value including `"external_file_id"` and `"external_file_name"` set from the file's `id` and `name` respectively using expressions.
   - Assign Pinecone API credentials (API key and HTTP header auth).

4. **Get File to Delete:**
   - Add HTTP Request node named `Get file to delete`.
   - Set method to GET.
   - URL: `https://prod-1-data.ke.pinecone.io/assistant/files/n8n-assistant`.
   - Add query parameter `filter` with JSON value filtering by `"external_file_id"` set to the updated file's id.
   - Use Pinecone API credentials.

5. **Delete File from Assistant:**
   - Add HTTP Request node named `Delete file from assistant`.
   - Set method to DELETE.
   - URL: `https://prod-1-data.ke.pinecone.io/assistant/files/n8n-assistant/{{ $json.files[0].id }}` (dynamic path).
   - Use Pinecone API credentials.

6. **Set File Data:**
   - Add Set node named `Set file data`.
   - Configure JSON output with:
     ```json
     {
       "id": "={{ $('File updated').item.json.id }}",
       "name": "={{ $('File updated').item.json.name }}"
     }
     ```
   - Connect output from `Delete file from assistant` to this node.

7. **Check File Status:**
   - Add HTTP Request node named `Check file status`.
   - Set method to GET.
   - URL: `https://prod-1-data.ke.pinecone.io/assistant/files/n8n-assistant/{{ $json.id }}`.
   - Use Pinecone API credentials.

8. **End if Status Available:**
   - Add Switch node named `End if status Available`.
   - Configure two outputs:
     - Output "Available" if `$json.status === "Available"`.
     - Output "Processing" if `$json.status === "Processing"`.

9. **Wait Node:**
   - Add Wait node named `Wait`.
   - Set wait time to 3 seconds.
   - Connect "Processing" output of Switch node to Wait node.
   - Connect Wait node back to `Check file status` to create a polling loop.

10. **Chat Input:**
    - Add LangChain Chat Trigger node named `Chat input`.
    - Configure webhook (auto-generated or specified).
    - No extra options to configure.

11. **AI Agent:**
    - Add LangChain Agent node named `AI Agent`.
    - Connect `Chat input` main output to `AI Agent` main input.

12. **OpenAI Chat Model:**
    - Add LangChain LM Chat OpenAI node named `OpenAI Chat Model`.
    - Set model to `gpt-4.1-mini` (or other GPT-4 variant).
    - Assign OpenAI API credentials.
    - Connect output to `AI Agent` via `ai_languageModel` input.

13. **Pinecone Assistant:**
    - Add LangChain MCP Client Tool node named `Pinecone Assistant`.
    - Set endpoint URL to `https://prod-1-data.ke.pinecone.io/mcp/assistants/n8n-assistant`.
    - Set authentication to Bearer Auth with Pinecone Bearer token credential.
    - Use HTTP streaming transport.
    - Connect output to `AI Agent` via `ai_tool` input.

14. **Conversation Memory:**
    - Add LangChain Memory Buffer Window node named `Conversation Memory`.
    - Use default settings.
    - Connect output to `AI Agent` via `ai_memory` input.

15. **Sticky Notes:**
    - Add Sticky Note nodes to document:
      - Block 1 labeled “1. Add new or update files from Google Drive to Pinecone Assistant”.
      - Block 2 labeled “2. Chat with your docs”.
      - Setup guide note with detailed instructions and links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow enables chat interaction with Google Drive documents (.docx, .json, .md, .txt, .pdf) using OpenAI’s GPT-4 and Pinecone Assistant for real-time context retrieval without needing to train custom LLMs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Project Purpose                                                                                                           |
| Prerequisites: Pinecone account with API key, GCP project with Google Drive API enabled, OpenAI account with API key.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Setup Requirements                                                                                                        |
| Pinecone Assistant setup: create an assistant named `n8n-assistant` in United States region, update nodes if a different name/region is used.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Pinecone Assistant Setup                                                                                                  |
| Google Drive OAuth2 credentials must be created in n8n and linked to Google Cloud Console with correct Client ID, Secret, and Redirect URI. Credential named `Google Drive account`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Google Drive OAuth2 Setup                                                                                                |
| Pinecone API key credential and Pinecone Bearer token credential must be created and linked in n8n for HTTP requests and MCP client tool nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Pinecone Credentials Setup                                                                                                |
| OpenAI API key credential must be created and linked for the OpenAI Chat Model node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | OpenAI Credentials Setup                                                                                                  |
| Files must be added to a Google Drive folder named `n8n-pinecone-demo` or update triggers accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Folder Naming Requirement                                                                                                |
| Chat input node’s webhook can be made publicly available, or custom chat interfaces can be developed to call it.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Chat Interface Customization                                                                                             |
| Support and community help are available via Pinecone Discord, Pinecone Forum, and GitHub repository issues.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Support Links                                                                                                            |
| [Pinecone Discord Community](https://discord.gg/tJ8V62S3sH)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Community Link                                                                                                           |
| [Pinecone Forum](https://community.pinecone.io/)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Community Link                                                                                                           |
| [GitHub Issues for n8n Templates](https://github.com/pinecone-io/n8n-templates/issues/new/choose)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Support Link                                                                                                            |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all current content policies with no illegal, offensive, or protected elements. All data processed is legal and public.

---