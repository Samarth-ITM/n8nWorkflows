Build a RAG system by uploading PDFs to the Google Gemini File Search Store

https://n8nworkflows.xyz/workflows/build-a-rag-system-by-uploading-pdfs-to-the-google-gemini-file-search-store-11197


# Build a RAG system by uploading PDFs to the Google Gemini File Search Store

### 1. Workflow Overview

This workflow implements a **Retrieval-Augmented Generation (RAG) system** using Google Gemini’s File Search API within n8n. It enables users to upload PDF files into a dedicated Google Gemini File Search Store, which acts as a private vector index for document search. Subsequently, users can interact with a chat interface that queries these indexed documents to receive context-aware answers.

The workflow is logically divided into three main blocks:

- **1.1 Store Creation and Configuration**  
  Handles manual creation of a Google Gemini File Search Store, which is the foundational storage for uploaded documents.

- **1.2 File Upload and Indexing**  
  Manages file uploads via a form submission, uploads the file to Google Gemini, and imports it into the previously created Search Store.

- **1.3 Conversational Query and Retrieval**  
  Listens for chat messages, retrieves relevant context from the Search Store using a LangChain Agent, and responds using the Google Gemini Chat Model, leveraging the Search Store tool for retrieval.

---

### 2. Block-by-Block Analysis

#### 2.1 Store Creation and Configuration

**Overview:**  
This block creates a new Google Gemini File Search Store via a manual trigger. The store acts as a private searchable index for uploaded documents.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Create Store (HTTP Request)  
- Sticky Note (with example store JSON)  
- Sticky Note1 (Step 1 explanation)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the store creation process manually.  
  - Inputs: None  
  - Outputs: Triggers the "Create Store" HTTP Request node.

- **Create Store**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Google Gemini API endpoint `/v1beta/fileSearchStores` to create a new File Search Store.  
  - Configuration:  
    - URL: `https://generativelanguage.googleapis.com/v1beta/fileSearchStores`  
    - Method: POST  
    - Headers: Content-Type application/json  
    - Body: JSON with `"displayName"` parameter (set to "XXX" placeholder).  
    - Query Parameter: API key (placeholder `"XXX"` to be replaced with actual key).  
  - Input: Triggered by manual trigger node  
  - Output: JSON response with File Search Store details (including resource name).  
  - Potential Failures: Auth errors (invalid API key), network timeouts, JSON parsing errors.  
  - Version: Uses HTTP Request node version 4.3.

- **Sticky Note**  
  - Provides example JSON of a created File Search Store with resource name and metadata.

- **Sticky Note1**  
  - Describes Step 1: creating a new Search Store manually via this block.

---

#### 2.2 File Upload and Indexing

**Overview:**  
This block receives files via a web form, uploads the file to Google Gemini, and imports it into the previously created Search Store to make it searchable.

**Nodes Involved:**  
- On form submission (Form Trigger)  
- Merge (Combine Form Data + Store Info)  
- Get Store (Set node providing Search Store resource name)  
- Upload File (HTTP Request)  
- Import to Store (HTTP Request)  
- Sticky Note2 (Step 2 explanation)  
- Sticky Note4 (Reminder to set store resource name)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Receives file uploads directly from users through a form with a single required file field labeled “File.”  
  - Configuration:  
    - Webhook ID for external access  
    - Form Title: “Test”  
    - Form Fields: single required file input  
  - Output: Form data including the uploaded file binary data.  
  - Failure modes: Missing file upload, invalid file types, webhook errors.

- **Merge**  
  - Type: Merge node, mode "combine all"  
  - Role: Combines the file upload data with the stored Search Store resource name from "Get Store" node.  
  - Inputs: Output from "On form submission" and "Get Store"  
  - Output: Merged JSON for downstream use.

- **Get Store**  
  - Type: Set  
  - Role: Provides the Google Gemini File Search Store resource name as a string (e.g., `"fileSearchStores/my-store-XXX"`).  
  - Output: JSON with key `"Search Store"` used downstream.  
  - Note: Must be manually updated with actual store resource name after store creation.

- **Upload File**  
  - Type: HTTP Request  
  - Role: Uploads the file binary data to Google Gemini’s file upload endpoint.  
  - Configuration:  
    - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`  
    - Method: POST  
    - Content-Type: binaryData (file upload)  
    - Query parameter: API key  
    - Headers: Content-Type application/json (may be inconsistent, but likely required)  
    - Input data field: `"File"` (from form submission)  
  - Output: JSON response with file metadata (name, id, etc.)  
  - Failure modes: Auth errors, file size limits, upload timeouts.

- **Import to Store**  
  - Type: HTTP Request  
  - Role: Imports the uploaded file into the Search Store, making it available for search.  
  - Configuration:  
    - URL built dynamically using store resource name from “Get Store” node:  
      `https://generativelanguage.googleapis.com/v1beta/{Search Store}:importFile`  
    - Method: POST  
    - Body: JSON with `"file_name"` set to the uploaded file’s name from `"Upload File"` output.  
    - Query parameter: API key  
    - Headers: Content-Type application/json  
  - Input: Triggered after "Upload File"  
  - Output: Confirmation of import operation.  
  - Failure modes: Incorrect store name, file not found, auth issues.

- **Sticky Note2**  
  - Describes Step 2: uploading and importing a file into the Search Store.

- **Sticky Note4**  
  - Reminds to set the correct store resource name in the “Get Store” node.

---

#### 2.3 Conversational Query and Retrieval

**Overview:**  
This block listens for chat messages, retrieves relevant documents from the Search Store using a LangChain Agent integrated with Google Gemini Chat Model, and returns context-aware responses.

**Nodes Involved:**  
- When chat message received (LangChain Chat Trigger)  
- Get Store1 (Set node with Search Store resource name)  
- Merge1 (Combines store info and chat trigger data)  
- Rag Agent (LangChain Agent node)  
- Google Gemini Chat Model (Language Model node)  
- Simple Memory (LangChain Memory Buffer)  
- SearchStore (HTTP Request Tool for file search)  
- Sticky Note3 (Step 3 explanation)  
- Sticky Note5 (General workflow description)

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Waits for incoming chat messages to start retrieval and response generation.  
  - Output: Chat message data.  
  - Failure modes: Webhook errors, message parsing errors.

- **Get Store1**  
  - Type: Set  
  - Role: Provides the Search Store resource name as a string for use in chat context. Similar to “Get Store” node but used in chat path.  
  - Must be updated manually with actual store resource name.

- **Merge1**  
  - Type: Merge node, combine mode  
  - Role: Combines chat trigger data with the Search Store resource name for downstream processing.

- **Rag Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates the retrieval and generation pipeline. Configured with a system message instructing it to always use the SearchStore tool for answering user queries.  
  - Inputs:  
    - AI language model (Google Gemini Chat Model)  
    - AI memory (Simple Memory)  
    - AI tool (SearchStore HTTP Request Tool)  
  - Failure modes: Agent internal errors, tool connection failures.

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model  
  - Role: Generates conversational responses informed by retrieved context.  
  - Credentials: Google Palm API credentials required.  
  - Failure modes: API quota issues, authentication errors, model latency.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context window to enable coherent multi-turn dialogue.  
  - Failure modes: Memory overflow, state inconsistencies.

- **SearchStore**  
  - Type: HTTP Request Tool  
  - Role: Calls Google Gemini content generation endpoint with a file search tool query.  
  - Configuration:  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`  
    - Method: POST  
    - Body: JSON containing the user question and references to the Search Store (resource name dynamically injected).  
    - Query parameter: API key  
  - Input: Receives query text from the agent; returns search-augmented content.  
  - Failure modes: API errors, invalid store name, timeout.

- **Sticky Note3**  
  - Explains Step 3: retrieval of context from Search Store during chat interaction.

- **Sticky Note5**  
  - Provides an extensive description of the entire workflow, setup instructions, and usage guidelines.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                        | Input Node(s)                  | Output Node(s)              | Sticky Note                                           |
|------------------------------|-----------------------------------|-------------------------------------|-------------------------------|-----------------------------|-------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Starts manual creation of Search Store | None                          | Create Store                |                                                       |
| Create Store                 | HTTP Request                      | Creates Google Gemini File Search Store | When clicking ‘Execute workflow’ | None                      |                                                       |
| Sticky Note                 | Sticky Note                      | Shows example JSON of created store | None                          | None                        |                                                       |
| Sticky Note1                | Sticky Note                      | Explains Step 1: Create Search Store | None                          | None                        |                                                       |
| On form submission           | Form Trigger                     | Receives uploaded files via form     | None                          | Merge                       |                                                       |
| Merge                       | Merge (combine all)               | Combines form data with store info   | On form submission, Get Store | Upload File                 |                                                       |
| Get Store                   | Set                              | Sets Search Store resource name       | None                          | Merge                       | "Set the Search Store created in STEP 1..."           |
| Upload File                 | HTTP Request                     | Uploads file to Google Gemini         | Merge                         | Import to Store             |                                                       |
| Import to Store             | HTTP Request                     | Imports uploaded file into store      | Upload File                   | None                        |                                                       |
| Sticky Note2                | Sticky Note                      | Explains Step 2: Upload and import file | None                          | None                        |                                                       |
| Sticky Note4                | Sticky Note                      | Reminder to set store resource name   | None                          | None                        |                                                       |
| When chat message received  | LangChain Chat Trigger           | Starts chat message processing        | None                          | Get Store1                  |                                                       |
| Get Store1                 | Set                              | Sets Search Store resource name for chat | When chat message received    | Merge1                      |                                                       |
| Merge1                     | Merge (combine all)               | Combines chat data with store info    | Get Store1, When chat message received | Rag Agent              |                                                       |
| Rag Agent                  | LangChain Agent                  | Retrieves context and generates answer | Merge1                       | None                        |                                                       |
| Google Gemini Chat Model    | LangChain LM Chat Google Gemini  | Generates chat responses               | Rag Agent                    | None                        |                                                       |
| Simple Memory              | LangChain Memory Buffer Window   | Maintains chat context window          | Rag Agent                    | None                        |                                                       |
| SearchStore                | HTTP Request Tool                | Searches files in Search Store         | Rag Agent                    | None                        |                                                       |
| Sticky Note3               | Sticky Note                      | Explains Step 3: Context retrieval in chat | None                          | None                        |                                                       |
| Sticky Note5               | Sticky Note                      | Full workflow detailed explanation    | None                          | None                        |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create HTTP Request Node to Create Search Store**  
   - Name: `Create Store`  
   - Type: HTTP Request (version 4.3)  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/fileSearchStores`  
   - Headers: `Content-Type: application/json`  
   - Body (JSON): `{ "displayName": "Your Store Name" }`  
   - Query Parameters: `key` set to your Google Gemini API key  
   - Connect: from `When clicking ‘Execute workflow’` node output.

3. **Create Set Node for Store Resource Name**  
   - Name: `Get Store`  
   - Type: Set  
   - Add string field `"Search Store"` with value as the resource name returned from step 2, e.g., `"fileSearchStores/my-store-XXX"`  
   - This value must be manually updated after creation.

4. **Create Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger (version 2.3)  
   - Configure form: Title “Test”, one required file field labeled “File”  
   - Webhook ID auto-generated for external file upload.

5. **Create Merge Node**  
   - Name: `Merge`  
   - Type: Merge (mode: combine all)  
   - Connect inputs: `On form submission` and `Get Store`  
   - Purpose: merge uploaded file data with store info.

6. **Create HTTP Request Node for File Upload**  
   - Name: `Upload File`  
   - Type: HTTP Request (version 4.3)  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/upload/v1beta/files`  
   - Query Parameter: `key` with your API key  
   - Content-Type: set to binaryData  
   - Headers: `Content-Type: application/json` (verify if required)  
   - Input Data Field Name: `File` (matches form upload)  
   - Connect input: from `Merge` node output.

7. **Create HTTP Request Node to Import File into Store**  
   - Name: `Import to Store`  
   - Type: HTTP Request (version 4.3)  
   - Method: POST  
   - URL: expression using store from `Get Store` node:  
     `https://generativelanguage.googleapis.com/v1beta/{{ $json["Search Store"] }}:importFile`  
   - Query Parameter: `key` with API key  
   - Body (JSON): `{ "file_name": "{{ $json.file.name }}" }` (file name from upload response)  
   - Headers: `Content-Type: application/json`  
   - Connect input: from `Upload File` node output.

8. **Create LangChain Chat Trigger Node**  
   - Name: `When chat message received`  
   - Type: LangChain Chat Trigger (version 1.4)  
   - No special parameters; webhook auto-generated.

9. **Create Set Node for Store Resource Name (Chat Path)**  
   - Name: `Get Store1`  
   - Type: Set  
   - Add string field `"Search Store"` with same resource name as in step 3.

10. **Create Merge Node**  
    - Name: `Merge1`  
    - Type: Merge (combine all)  
    - Connect inputs: `Get Store1` and `When chat message received`.

11. **Create LangChain Agent Node**  
    - Name: `Rag Agent`  
    - Type: LangChain Agent (version 3)  
    - Parameters:  
      - System Message: `"You are a helpful assistant. For every question you are asked, always search with the SearchStore tool"`  
    - Connect input: `Merge1` output.

12. **Create Google Gemini Chat Model Node**  
    - Name: `Google Gemini Chat Model`  
    - Type: LangChain LM Chat Google Gemini (version 1)  
    - Credentials: configure Google Palm API credentials here.  
    - Connect as AI Language Model input to `Rag Agent`.

13. **Create Simple Memory Node**  
    - Name: `Simple Memory`  
    - Type: LangChain Memory Buffer Window (version 1.3)  
    - Connect as AI Memory input to `Rag Agent`.

14. **Create HTTP Request Tool Node for SearchStore**  
    - Name: `SearchStore`  
    - Type: HTTP Request Tool (version 4.3)  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`  
    - Query Parameter: API key  
    - Body (JSON):  
      ```json
      {
        "contents": [
          {
            "parts": [
              {
                "text": "{{ $fromAI(\"query\",\"the question the user needs an answer to\") }}"
              }
            ]
          }
        ],
        "tools": [
          {
            "file_search": {
              "file_search_store_names": [
                "{{ $json['Search Store'] }}"
              ]
            }
          }
        ]
      }
      ```  
    - Connect as AI Tool input to `Rag Agent`.

15. **Connect Nodes Appropriately**  
    - `When clicking ‘Execute workflow’` → `Create Store`  
    - `On form submission` + `Get Store` → `Merge` → `Upload File` → `Import to Store`  
    - `When chat message received` → `Get Store1` + `When chat message received` → `Merge1` → `Rag Agent`  
    - `Google Gemini Chat Model` → `Rag Agent` (AI Language Model input)  
    - `Simple Memory` → `Rag Agent` (AI Memory input)  
    - `SearchStore` → `Rag Agent` (AI Tool input)

16. **API Key Management**  
    - Replace all `"XXX"` placeholders for API keys in HTTP Request nodes with your valid Google Gemini API key.  
    - Set Google Palm API credentials in the Google Gemini Chat Model node.

17. **Manual Steps**  
    - After creating the store once, copy the returned resource name into both `Get Store` and `Get Store1` nodes before enabling the workflow.

18. **Activation**  
    - Activate the workflow for the form and chat webhook URLs to become available for external usage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow implements a fully automated RAG system combining Google Gemini File Search API with LangChain agents to build scalable document question answering inside n8n. Users upload files via a form, which are indexed in a private Search Store. Chat queries then automatically retrieve relevant document passages to generate context-aware answers.                                                                                                                                                                                                                                                                                            | Detailed conceptual overview provided in Sticky Note5 node.                                                |
| After running the "Create Store" step manually, the returned store resource name must be updated in the "Get Store" and "Get Store1" nodes to link the workflow to the correct Search Store.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Highlighted in Sticky Note4 and workflow instructions.                                                     |
| Form submission node exposes a webhook URL that can be embedded or called from external user interfaces to allow file uploads directly into the Search Store.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Configuration in "On form submission" node.                                                                |
| Chat trigger node exposes a webhook URL for integrating chat interfaces, enabling conversational queries against the uploaded documents.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Configuration in "When chat message received" node.                                                        |
| The Google Gemini Chat Model node requires Google Palm API credentials configured in n8n credentials manager.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Credential setup step.                                                                                       |
| API keys used in HTTP Request nodes must have sufficient permission scopes for File Search Store creation, file upload, import, and content generation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | API key management best practices.                                                                          |
| This workflow was tested with Google Gemini API version v1beta endpoints and n8n nodes versions as specified per node. Compatibility with future API versions may require adjustments.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Version-specific notes in node analysis.                                                                   |