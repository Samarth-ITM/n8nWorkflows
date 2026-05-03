Build an AI IT Support Agent with Azure Search, Entra ID & Jira

https://n8nworkflows.xyz/workflows/build-an-ai-it-support-agent-with-azure-search--entra-id---jira-4560


# Build an AI IT Support Agent with Azure Search, Entra ID & Jira

---

# Reference Document for n8n Workflow:  
**Build an AI IT Support Agent with Azure Search, Entra ID & Jira**

---

## 1. Workflow Overview

This workflow builds an AI-powered IT support agent for Acme Corporation that interacts with users via chat, leverages Azure AI Search for company knowledge retrieval, integrates with Microsoft Entra ID (Azure Active Directory) for user management (e.g., password resets), and creates Jira tickets for issue tracking. It supports uploading internal documentation to an Azure AI Search vector index, semantic search for relevant documents, and AI-driven responses with conversational memory.

The workflow is organized into four main logical blocks:

- **1.1 User Interaction and AI Agent Logic**  
  Handles chat message reception, AI reasoning with memory, and invoking AI language models. Also manages calls to Microsoft Entra ID and Jira tools based on AI decisions.

- **1.2 Document Upload and Azure AI Search Index Creation**  
  Supports uploading internal docs (.txt, .md) via a form, processing and embedding them, and creating/updating the Azure AI Search vector index to build the knowledge base.

- **1.3 Semantic Search Pipeline**  
  Receives search queries, generates embeddings, retrieves Azure AI Search admin keys, and queries the Azure AI Search index using vector and semantic search.

- **1.4 Microsoft Entra ID and Jira Integration**  
  Executes user account queries in Microsoft Entra ID and resets passwords or creates Jira tickets, triggered by the AI agent.

---

## 2. Block-by-Block Analysis

### 2.1 User Interaction and AI Agent Logic

**Overview:**  
This block is the main entry point for user interaction. It triggers on new chat messages, sets Azure config fields, and passes chat input to an AI agent node that uses conversational memory and AI language models (Google Gemini) to generate responses. The agent can invoke tools like Microsoft Entra ID user queries, password resets, Jira ticket creation, and vector store search.

**Nodes Involved:**  
- When chat message received  
- Set Common Fields2  
- IT Support Agent  
- Simple Memory  
- Google Gemini Chat Model  
- Query Microsoft Entra ID Users  
- Reset User Password  
- Create Jira Ticket  
- Invoke Query Vector Store Webhook

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Workflow entry triggered by chat input from users  
  - Config: Public webhook with n8n user authentication; initial greeting message configured.  
  - Input: External chat message  
  - Output: Chat input text, session ID for memory  
  - Failure cases: Auth errors, webhook connectivity issues  

- **Set Common Fields2**  
  - Type: Set  
  - Role: Defines Azure subscription ID, resource group, AI search service/index names used downstream  
  - Parameters: Placeholder strings for Azure config variables  
  - Failure cases: Misconfiguration leads to failed Azure API calls  

- **IT Support Agent**  
  - Type: Langchain Agent  
  - Role: Core AI logic node using memory, reasoning, and tools to answer user queries  
  - Config:  
    - System message defines behavior as IT support agent for Acme Corp  
    - Tool usage includes vector store search, memory, Microsoft Entra ID, Jira  
    - Prompt input is user chat input from trigger node  
  - Inputs: Chat input text, memory context, AI language model output (Google Gemini)  
  - Outputs: AI-generated response, tool invocations (e.g., Jira ticket, password reset)  
  - Failure cases: AI model errors, expression evaluation failures, tool API failures  
  - Notes: Avoids assumptions, requires vector store lookup before querying Entra ID  

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversation context by storing recent chat history (20 messages) keyed by session ID  
  - Input: Session ID from chat trigger  
  - Output: Memory context to agent  
  - Failure: Memory overflow or session ID issues  

- **Google Gemini Chat Model**  
  - Type: Langchain LM Chat Google Gemini  
  - Role: AI language model generating responses  
  - Config: Uses Google Gemini 2.5 flash preview model  
  - Credentials: Google PaLM API key  
  - Input: Text prompt from agent  
  - Output: Language model completion to agent  
  - Failure: API errors, rate limits  

- **Query Microsoft Entra ID Users**  
  - Type: HTTP Request Tool  
  - Role: Queries Microsoft Entra ID user accounts using AI-generated filter parameters  
  - Config: Calls Graph API /users with $filter query from AI  
  - Credentials: Microsoft Entra OAuth2  
  - Input: Filter expression from AI agent  
  - Output: User data JSON for agent processing  
  - Failure: Auth errors, invalid filters, API limits  

- **Reset User Password**  
  - Type: HTTP Request Tool  
  - Role: Resets user password in Microsoft Entra ID per AI request  
  - Config: PATCH request to Graph API /users/{username} with new password JSON  
  - Credentials: Microsoft Entra OAuth2  
  - Input: Username from AI agent  
  - Output: Success/failure response to agent  
  - Failure: Auth errors, invalid usernames, API errors  

- **Create Jira Ticket**  
  - Type: Jira Tool  
  - Role: Creates Jira tasks for IT issues as directed by AI agent  
  - Config: Project ID 10000, issue type "Task", summary and description from AI overrides  
  - Credentials: Jira Software Cloud API  
  - Input: AI-generated ticket data  
  - Output: Created Jira issue data  
  - Failure: Auth errors, invalid project/issue type, API errors  

- **Invoke Query Vector Store Webhook**  
  - Type: HTTP Request Tool  
  - Role: Calls internal webhook to query Azure AI Search index for relevant docs  
  - Config: POST to semantic search webhook URL with Azure config and user question  
  - Input: Azure config and chat input text  
  - Output: Search results to AI agent for response generation  
  - Failure: Webhook unavailability, invalid parameters  

---

### 2.2 Document Upload and Azure AI Search Index Creation

**Overview:**  
This block enables uploading internal documentation files (.txt, .md), processing them to extract text content, generating embeddings, and uploading the data to an Azure AI Search vector index for semantic search.

**Nodes Involved:**  
- On Knowledge Upload  
- Split Out Binary Files  
- Prep Content  
- Get Embeddings  
- Set Common Fields1  
- Get Azure AI Search Admin Key1  
- Upload Embedding to Vector Store

**Node Details:**

- **On Knowledge Upload**  
  - Type: Form Trigger  
  - Role: Exposes a file upload form for internal documentation  
  - Config: Accepts .txt and .md files; required field  
  - Output: Binary files from form submission  
  - Failure: Invalid file types, upload errors  

- **Split Out Binary Files**  
  - Type: Split Out  
  - Role: Splits uploaded files array into individual items for processing  
  - Config: Splits on binary field "Documents"  
  - Output: One item per uploaded document  
  - Failure: Empty input, binary data missing  

- **Prep Content**  
  - Type: Code (JavaScript)  
  - Role: Decodes base64 content, extracts file metadata (filename), and generates SHA-256 hash ID  
  - Output: JSON with id (hash), title (filename), content (decoded text), category ("General")  
  - Failure: Base64 decoding errors, no binary data, code exceptions  

- **Get Embeddings**  
  - Type: HTTP Request  
  - Role: Calls OpenAI embedding API ("text-embedding-ada-002") to get vector representation of document content  
  - Config: POST request with document text in body  
  - Credentials: OpenAI API key  
  - Output: Embedding vector JSON  
  - Failure: API errors, invalid content, rate limits  

- **Set Common Fields1**  
  - Type: Set  
  - Role: Defines Azure subscription ID, resource group, AI search service/index names for document upload  
  - Config: Placeholder Azure config variables  
  - Failure: Misconfiguration leads to failed Azure API calls  

- **Get Azure AI Search Admin Key1**  
  - Type: HTTP Request  
  - Role: Retrieves admin API key for Azure AI Search service to authenticate index updates  
  - Config: POST request to Azure management endpoint  
  - Credentials: Microsoft Azure OAuth2  
  - Output: JSON with primary key  
  - Failure: Auth errors, invalid service name  

- **Upload Embedding to Vector Store**  
  - Type: HTTP Request  
  - Role: Sends document data and embedding vector to Azure AI Search index using "mergeOrUpload" action  
  - Config: POST request to Azure Search index docs endpoint with JSON body including id, title, content, category, and contentVector embedding  
  - Headers: API key from admin key node  
  - Failure: Auth errors, incorrect JSON, API errors  

---

### 2.3 Semantic Search Pipeline

**Overview:**  
This block handles user search queries by generating embeddings, retrieving the Azure AI Search admin key, and querying the Azure vector index using semantic and vector search to find relevant internal documentation.

**Nodes Involved:**  
- Semantic Search (Webhook)  
- Get Embeddings1  
- Get Azure AI Search Admin Key2  
- Query Azure AI Search Index

**Node Details:**

- **Semantic Search**  
  - Type: Webhook  
  - Role: Receives POST requests with user query and Azure config to trigger search flow  
  - Input: JSON body with user search content and Azure details  
  - Output: Passes input to embedding generation node  
  - Failure: Webhook unavailability, invalid payload  

- **Get Embeddings1**  
  - Type: HTTP Request  
  - Role: Calls OpenAI embedding API to create vector of user search input  
  - Config: POST with user query text  
  - Credentials: OpenAI API  
  - Output: Embedding vector for query  
  - Failure: API errors, empty input  

- **Get Azure AI Search Admin Key2**  
  - Type: HTTP Request  
  - Role: Retrieves Azure AI Search admin key for querying the index  
  - Config: POST to Azure management API with subscription, resource group, service name from webhook input  
  - Credentials: Microsoft Azure OAuth2  
  - Output: Admin key JSON  
  - Failure: Auth errors, wrong parameters  

- **Query Azure AI Search Index**  
  - Type: HTTP Request  
  - Role: Performs semantic and vector search on Azure AI index using user query and embedding vector  
  - Config: POST search request with semantic config, vectorQueries (embedding), selecting relevant document fields, top 5 results  
  - Headers: API key from admin key node  
  - Output: Search results JSON with matching documents  
  - Failure: Auth errors, malformed requests, API errors  

---

### 2.4 Microsoft Entra ID and Jira Integration

**Overview:**  
This block provides nodes for interacting with Microsoft Entra ID to query users and reset passwords, and for creating Jira tickets. These nodes are invoked by the AI agent when appropriate based on user requests.

**Nodes Involved:**  
- Query Microsoft Entra ID Users (already detailed in 2.1)  
- Reset User Password (already detailed in 2.1)  
- Create Jira Ticket (already detailed in 2.1)

**Node Details:**  
All nodes are HTTP or Jira API integrations secured with OAuth2 credentials. They depend on AI agent inputs and provide operation results back to the agent. Potential failures include authentication issues, API limits, invalid parameters, and network errors.

---

## 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                           | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                          |
|--------------------------------|---------------------------------------|-----------------------------------------|-------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received      | Langchain Chat Trigger                 | Workflow entry point for user chat      | -                             | Set Common Fields2               | This node triggers the workflow when a new chat message is received. It serves as the main entry point for user interaction with the IT support agent. |
| Set Common Fields2              | Set                                   | Define Azure config for AI Search       | When chat message received     | IT Support Agent                |                                                                                                    |
| IT Support Agent                | Langchain Agent                       | AI reasoning and tool orchestration      | Set Common Fields2, Simple Memory, Google Gemini Chat Model, Query Microsoft Entra ID Users, Reset User Password, Create Jira Ticket, Invoke Query Vector Store Webhook | Google Gemini Chat Model, Simple Memory, Query Microsoft Entra ID Users, Reset User Password, Create Jira Ticket, Invoke Query Vector Store Webhook | Defines how the AI agent behaves, combining memory, LLM, and tools for accurate IT support responses. |
| Simple Memory                  | Langchain Memory Buffer Window         | Maintain conversation context            | When chat message received     | IT Support Agent                | This node manages conversational memory by tracking previous messages.                             |
| Google Gemini Chat Model       | Langchain LM Chat Google Gemini        | AI language model generating responses  | IT Support Agent               | IT Support Agent                |                                                                                                    |
| Query Microsoft Entra ID Users | HTTP Request Tool                      | Query Entra ID user accounts             | IT Support Agent               | IT Support Agent                | Sends filtered queries to Microsoft Entra ID to find user accounts matching AI criteria.           |
| Reset User Password            | HTTP Request Tool                      | Reset user password in Microsoft Entra  | IT Support Agent               | IT Support Agent                | Resets Microsoft Entra ID user password. Always informs user of the new password.                   |
| Create Jira Ticket             | Jira Tool                             | Create Jira task for IT issues           | IT Support Agent               | IT Support Agent                | Creates Jira tickets for escalated user issues.                                                    |
| Invoke Query Vector Store Webhook | HTTP Request Tool                  | Query Azure AI Search index via webhook | IT Support Agent               | IT Support Agent                | Helps the agent find relevant internal documentation by querying the vector store.                  |
| On Knowledge Upload            | Form Trigger                          | Upload internal docs for knowledge base | -                             | Split Out Binary Files          | Provides file upload form for documentation in .txt or .md format.                                 |
| Split Out Binary Files         | Split Out                            | Split uploaded files into individual docs | On Knowledge Upload            | Prep Content                   | Separates uploaded files for processing.                                                           |
| Prep Content                  | Code                                 | Decode files and prepare metadata        | Split Out Binary Files          | Get Embeddings                 | Processes docs: decodes base64, extracts file info, generates unique ID hash.                      |
| Get Embeddings                | HTTP Request                         | Generate text embeddings from OpenAI     | Prep Content                  | Set Common Fields1              | Calls OpenAI embedding API for document text.                                                      |
| Set Common Fields1            | Set                                  | Define Azure config for document upload  | Get Embeddings                | Get Azure AI Search Admin Key1  |                                                                                                    |
| Get Azure AI Search Admin Key1 | HTTP Request                        | Retrieve Azure AI Search admin key        | Set Common Fields1            | Upload Embedding to Vector Store | Needed to authenticate Azure Search index updates.                                                 |
| Upload Embedding to Vector Store | HTTP Request                      | Upload docs + embeddings to Azure AI Search | Get Azure AI Search Admin Key1 | -                             | Sends vector data to Azure AI Search index.                                                        |
| Semantic Search               | Webhook                             | Entry point for semantic search queries  | -                             | Get Embeddings1                | Receives POST with user search query and Azure config.                                             |
| Get Embeddings1              | HTTP Request                         | Generate embedding vector for search query | Semantic Search              | Get Azure AI Search Admin Key2  | Calls OpenAI embedding API for user query text.                                                    |
| Get Azure AI Search Admin Key2 | HTTP Request                      | Retrieve admin key for Azure Search query | Get Embeddings1              | Query Azure AI Search Index     | Authorizes querying Azure AI Search index.                                                         |
| Query Azure AI Search Index  | HTTP Request                         | Query Azure AI Search index for docs      | Get Azure AI Search Admin Key2 | -                             | Returns top matching documents based on semantic and vector search.                                |
| Set Common Fields             | Set                                  | Define Azure config for AI Search service creation | When clicking ‘Test workflow’ | Create Azure AI Search Service |                                                                                                    |
| When clicking ‘Test workflow’ | Manual Trigger                      | Manual start for testing workflow         | -                             | Set Common Fields               |                                                                                                    |
| Create Azure AI Search Service | HTTP Request                      | Create Azure AI Search service with semantic config | Set Common Fields             | Enable Semantic Search          | Requires basic pricing tier for semantic search support.                                           |
| Enable Semantic Search        | HTTP Request                      | Enable semantic search on Azure Search service | Create Azure AI Search Service | Get Azure AI Search Admin Key   | Enables semantic search features after service creation.                                           |
| Get Azure AI Search Admin Key | HTTP Request                      | Retrieve admin key for Azure AI Search service | Enable Semantic Search        | Create Azure AI Vector Index    | Needed for index creation authorization.                                                           |
| Create Azure AI Vector Index  | HTTP Request                      | Create Azure AI Search vector index with semantic config | Get Azure AI Search Admin Key | -                             | Defines index schema, vector search algorithms, and semantic fields.                               |
| Sticky Note                   | Sticky Note                       | Section header: Uploading Docs to Azure AI Search Index | -                             | -                             | ## 2. Uploading Docs to Azure AI Search Index                                                      |
| Sticky Note1                  | Sticky Note                       | Section header: Create the Azure AI Search Index | -                             | -                             | ## 1. Create the Azure AI Search Index                                                             |
| Sticky Note2                  | Sticky Note                       | Section header: Vector Store Search       | -                             | -                             | ## Vector Store Search                                                                             |
| Sticky Note3                  | Sticky Note                       | Section header: Agent                      | -                             | -                             | ## Agent                                                                                           |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger for User Chat:**  
   - Add **Langchain Chat Trigger** node named `When chat message received`  
   - Set to public, authentication: `n8nUserAuth`  
   - Initial message: "Hello, what can I help you with today?"  

2. **Set Azure Config for Agent:**  
   - Add **Set** node `Set Common Fields2` connected to chat trigger  
   - Define variables:  
     - `azure_ai_search_service_name` (string)  
     - `azure_ai_search_location` (string)  
     - `azure_ai_search_index_name` (string)  
     - `azure_resource_group` (string)  
     - `azure_subsription_id` (string)  

3. **Add AI Agent Node:**  
   - Add **Langchain Agent** node `IT Support Agent` after `Set Common Fields2`  
   - Configure system message with behavior rules for IT support agent (as described)  
   - Set prompt input: `={{ $('When chat message received').item.json.chatInput }}`  
   - Configure tools: memory, vector store search webhook, Microsoft Entra ID user query, password reset, Jira ticket creation  

4. **Add Conversational Memory:**  
   - Add **Langchain Memory Buffer Window** node `Simple Memory`  
   - Session key: `={{ $('When chat message received').item.json.sessionId }}`  
   - Context window length: 20 messages  
   - Connect AI agent to use memory  

5. **Add Google Gemini Model:**  
   - Add **Langchain LM Chat Google Gemini** node `Google Gemini Chat Model`  
   - Model: `models/gemini-2.5-flash-preview-05-20`  
   - Set credentials for Google PaLM API  
   - Connect to AI agent as language model  

6. **Configure Microsoft Entra ID User Query Node:**  
   - Add **HTTP Request Tool** node `Query Microsoft Entra ID Users`  
   - URL: `https://graph.microsoft.com/v1.0/users`  
   - Method: GET with `$filter` query parameter from AI overrides  
   - OAuth2 credentials for Microsoft Entra ID  
   - Connect AI agent tool input/output  

7. **Configure Microsoft Entra ID Password Reset Node:**  
   - Add **HTTP Request Tool** node `Reset User Password`  
   - Method: PATCH to `https://graph.microsoft.com/v1.0/users/{{ $fromAI('entra_id_username') }}`  
   - JSON body with password profile (forceChangePasswordNextSignIn: true, password: "NewPassword123!")  
   - OAuth2 credentials for Microsoft Entra ID  
   - Connect AI agent tool I/O  

8. **Configure Jira Ticket Creation Node:**  
   - Add **Jira Tool** node `Create Jira Ticket`  
   - Project ID: 10000 (IT Support Tickets)  
   - Issue Type: Task (ID 10003)  
   - Summary and description from AI overrides  
   - Jira Software Cloud API credentials  
   - Connect AI agent tool I/O  

9. **Configure Vector Store Query Webhook Invocation:**  
   - Add **HTTP Request Tool** node `Invoke Query Vector Store Webhook`  
   - URL: semantic search webhook URL (internal)  
   - POST body includes Azure config and user chat input  
   - Connect AI agent tool I/O  

10. **Build Document Upload Pipeline:**  
    - Add **Form Trigger** node `On Knowledge Upload`  
    - Configure form with file field accepting `.txt,.md` files required  
    - Connect to **Split Out Binary Files** node `Split Out Binary Files` to separate files  

11. **Decode and Prep Document Content:**  
    - Add **Code** node `Prep Content`  
    - JavaScript to base64 decode, extract filename, generate SHA-256 hash ID, and prepare JSON with id, title, content, category = "General"  

12. **Generate Document Embeddings:**  
    - Add **HTTP Request** node `Get Embeddings`  
    - POST to OpenAI embedding API with body: input = document content, model = "text-embedding-ada-002"  
    - Set OpenAI API credentials  

13. **Set Azure Config for Upload:**  
    - Add **Set** node `Set Common Fields1`  
    - Define Azure subscription ID, resource group, AI search service/index name  

14. **Get Azure AI Search Admin Key for Upload:**  
    - Add **HTTP Request** node `Get Azure AI Search Admin Key1`  
    - POST to Azure management endpoint to list admin keys  
    - Use Microsoft OAuth2 credentials  

15. **Upload Documents and Embeddings to Azure Search:**  
    - Add **HTTP Request** node `Upload Embedding to Vector Store`  
    - POST to Azure Search index docs endpoint  
    - JSON body uses `mergeOrUpload` to add document with id, title, content, category, contentVector embedding  
    - Authorization header: admin key from previous node  

16. **Create Azure AI Search Service (Optional Initialization):**  
    - Add **Manual Trigger** node `When clicking ‘Test workflow’` for manual testing  
    - Add **Set** node `Set Common Fields` for Azure config  
    - Add **HTTP Request** `Create Azure AI Search Service` to create search service with semantic config (basic SKU)  
    - Add **HTTP Request** `Enable Semantic Search` to enable semantic search features  
    - Add **HTTP Request** `Get Azure AI Search Admin Key` to get admin key  
    - Add **HTTP Request** `Create Azure AI Vector Index` to create index with fields and vector search config  

17. **Setup Semantic Search Pipeline Webhook:**  
    - Add **Webhook** node `Semantic Search`  
    - POST path for receiving search queries with Azure config and user query  
    - Connect to **Get Embeddings1** (OpenAI embedding call for user query)  
    - Connect to **Get Azure AI Search Admin Key2** (retrieve admin key for querying)  
    - Connect to **Query Azure AI Search Index** (search Azure index with semantic config and vector queries)  

18. **Credentials Setup:**  
    - Google PaLM API credential for Google Gemini Chat Model  
    - OpenAI API credential for embedding calls  
    - Microsoft Azure OAuth2 credential for Azure management API calls  
    - Microsoft Entra ID OAuth2 credential for Graph API calls  
    - Jira Software Cloud OAuth2 credential for Jira operations  

---

## 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses Azure AI Search semantic search with vector search capabilities requiring a basic pricing tier. | Azure AI Search documentation: https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search |
| The AI agent uses a strict behavior prompt to avoid assumptions and inventing information.                      | Encourages accurate, concise IT support responses based on vector store lookup.                 |
| The document upload pipeline supports only plain text (.txt) and markdown (.md) files for knowledge ingestion.  | Can be extended to other formats with additional processing nodes.                              |
| Microsoft Entra ID API calls require OAuth2 credentials with sufficient permissions to query users and reset passwords. | See Microsoft Graph API docs: https://learn.microsoft.com/en-us/graph/api/resources/users?view=graph-rest-1.0 |
| Jira integration requires Jira Software Cloud OAuth2 setup with appropriate project and issue type access.      | Jira API docs: https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/               |
| The vector store querying is done via an internal webhook to decouple query logic from the AI agent node.       | Ensure webhook URL is correctly configured and secured.                                        |

---

**Disclaimer:**  
The text provided is derived solely from an automated workflow created with n8n, an integration and automation tool. This process strictly respects all applicable content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and public.

---