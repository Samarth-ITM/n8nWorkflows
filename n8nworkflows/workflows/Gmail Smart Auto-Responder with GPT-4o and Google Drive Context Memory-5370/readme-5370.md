Gmail Smart Auto-Responder with GPT-4o and Google Drive Context Memory

https://n8nworkflows.xyz/workflows/gmail-smart-auto-responder-with-gpt-4o-and-google-drive-context-memory-5370


# Gmail Smart Auto-Responder with GPT-4o and Google Drive Context Memory

---

### 1. Workflow Overview

This workflow enables an intelligent Gmail auto-responder powered by GPT-4o and Google Drive-based user profile context memory. It is designed for professionals or users who want automated, context-aware, and polite email replies that leverage both the user's profile and full email thread history to provide relevant, concise responses.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Detect new emails in Gmail excluding self-sent messages.
- **1.2 Context Retrieval:** Download user profile document from Google Drive and fetch the full email thread.
- **1.3 Context Processing:** Load and split both profile and thread documents into manageable text chunks.
- **1.4 Embedding & Storage:** Create embeddings from the text chunks and store them in an in-memory vector store keyed by the email thread ID.
- **1.5 Context Preparation:** Prepare a textual prompt combining incoming email details and instructions for the AI agent.
- **1.6 AI Agent Interaction:** Use a GPT-4o-based AI agent with memory and vector retrieval tools to generate an email reply or decide no reply is needed.
- **1.7 Reply Handling:** Condition to check if a reply is required, then create a Gmail draft for the reply.
- **1.8 Error Handling:** Centralized error logging and handling using a code node.
- **1.9 Workflow Instructions:** A sticky note node contains setup instructions and workflow overview for user reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Monitors Gmail inbox for new incoming emails excluding those sent by the user themselves, polling every minute.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**  
  - **Gmail Trigger**  
    - Type: Gmail Trigger (event-based trigger node)  
    - Configuration: Poll every minute; filter query excludes emails from self (`-from:me`)  
    - Expressions: None (static filter query)  
    - Input: None (trigger node)  
    - Output: Emits new incoming email JSON, including threadId, subject, headers, and email body  
    - Version Requirements: Gmail OAuth2 credentials required  
    - Edge Cases: Gmail API rate limits, OAuth token expiration, no new emails  
    - Sub-workflow: None

#### 2.2 Context Retrieval

- **Overview:**  
Downloads the user profile document from Google Drive and fetches the full email thread from Gmail using the thread ID.

- **Nodes Involved:**  
  - Google Drive - Download Profile  
  - Gmail - Fetch Thread

- **Node Details:**  
  - **Google Drive - Download Profile**  
    - Type: Google Drive (download operation)  
    - Configuration: Downloads file specified by `YOUR_PROFILE_DOC_FILE_ID` parameter  
    - Input: Triggered by Gmail Trigger output  
    - Output: File content (profile document)  
    - Credentials: Google Drive OAuth2 required  
    - Edge Cases: Incorrect file ID, permission errors, OAuth token expiration, file not found  
  - **Gmail - Fetch Thread**  
    - Type: Gmail (getAll operation for threads)  
    - Configuration: Fetches all messages in the thread specified by the incoming email's `threadId`  
    - Input: Triggered by Gmail Trigger output  
    - Output: JSON array of all messages in the thread  
    - Credentials: Gmail OAuth2 required  
    - Edge Cases: Thread ID invalid, rate limits, OAuth errors  

#### 2.3 Context Processing

- **Overview:**  
Loads the profile document and email thread messages as JSON data into document loader nodes, then splits the text into chunks for embedding.

- **Nodes Involved:**  
  - Profile Document Loader  
  - Thread Document Loader  
  - Profile Text Splitter  
  - Thread Text Splitter

- **Node Details:**  
  - **Profile Document Loader**  
    - Type: Langchain Document Loader (default JSON loader)  
    - Configuration: Loads downloaded profile content as JSON, with metadata type "user_profile"  
    - Input: Output of Google Drive Download Profile  
    - Output: Structured document data for embedding  
  - **Thread Document Loader**  
    - Type: Langchain Document Loader (default JSON loader)  
    - Configuration: Loads the thread messages JSON array, metadata type "email_thread"  
    - Input: Output of Gmail Fetch Thread  
    - Output: Structured document data for embedding  
  - **Profile Text Splitter**  
    - Type: Langchain Recursive Character Text Splitter  
    - Configuration: 1000 characters per chunk, 100 overlap  
    - Input: Profile Document Loader output  
    - Output: Chunks of profile text for embedding  
  - **Thread Text Splitter**  
    - Type: Langchain Recursive Character Text Splitter  
    - Configuration: 2000 characters per chunk, 200 overlap (larger chunks for thread)  
    - Input: Thread Document Loader output  
    - Output: Chunks of thread text for embedding  

- **Edge Cases:**  
  - Large profile or thread documents causing processing delays or token overflows  
  - Malformed JSON in documents  
  - Empty profile or thread content  

#### 2.4 Embedding & Storage

- **Overview:**  
Converts text chunks into vector embeddings using OpenAI embeddings, then inserts them into an in-memory vector store keyed by the thread ID.

- **Nodes Involved:**  
  - OpenAI Embeddings  
  - Simple Vector Store

- **Node Details:**  
  - **OpenAI Embeddings**  
    - Type: Langchain OpenAI embeddings node  
    - Configuration: Uses default OpenAI embedding model (no special options)  
    - Input: Output of both text splitters (merged via upstream connections)  
    - Output: Vector embeddings for each chunk  
    - Credentials: OpenAI API key required  
    - Edge Cases: API rate limits, invalid credentials, network issues  
  - **Simple Vector Store**  
    - Type: Langchain In-memory Vector Store  
    - Configuration: Insert mode; memory key dynamically constructed as `email_context_<threadId>`  
    - Input: Embeddings from OpenAI Embeddings node  
    - Output: Confirms insertion, prepares context for agent  
    - Edge Cases: Memory limit overflow, concurrency conflicts  

#### 2.5 Context Preparation

- **Overview:**  
Prepares a textual prompt containing the incoming email details and instructions for the AI agent, embedding references to the vector store context.

- **Nodes Involved:**  
  - Prepare Agent Context

- **Node Details:**  
  - **Prepare Agent Context**  
    - Type: Set Node (string assignment)  
    - Configuration: Assigns a multi-line string variable named `text` containing:  
      - Incoming email subject, sender, and body (HTML)  
      - Instructions to avoid replying to newsletters/spam, generate professional replies if needed, and use vector store knowledge  
    - Input: Output from Simple Vector Store node (context ready)  
    - Output: JSON with `text` field for AI agent input  
    - Edge Cases: Expression errors if fields missing, HTML encoding issues  

#### 2.6 AI Agent Interaction

- **Overview:**  
Uses the Langchain agent pattern with GPT-4o-mini chat model, vector retrieval tool, and memory buffer to generate the email reply or a "NO_REPLY_NEEDED" marker.

- **Nodes Involved:**  
  - Vector Store Retriever  
  - Simple Memory  
  - OpenAI Chat Model  
  - Email Reply Agent

- **Node Details:**  
  - **Vector Store Retriever**  
    - Type: Vector Store In-Memory (retrieve mode as tool)  
    - Configuration: Retrieves up to 5 relevant context chunks from vector store keyed by current threadId  
    - Input: None explicitly (memory key constructed dynamically)  
    - Output: Supplies context to Email Reply Agent as a tool named "email_context"  
    - Edge Cases: Empty vector store, retrieval errors  
  - **Simple Memory**  
    - Type: Memory Buffer Window  
    - Configuration: Session key based on threadId, max token limit 4000, maintains conversation history  
    - Input: Connects as `ai_memory` to Email Reply Agent  
    - Output: Provides conversational memory to agent  
  - **OpenAI Chat Model**  
    - Type: Langchain Chat Model Node  
    - Configuration: Uses GPT-4o-mini model with temperature 0.3 for deterministic, professional tone  
    - Input: Connected as `ai_languageModel` to Email Reply Agent  
    - Output: Text output of AI-generated reply or NO_REPLY_NEEDED  
    - Credentials: OpenAI API key required  
    - Edge Cases: API limits, network issues  
  - **Email Reply Agent**  
    - Type: Langchain Agent Node  
    - Configuration: Takes `text` input from Prepare Agent Context, uses system message guiding polite, context-aware email replies, implements prompt with user profile and thread context via tools and memory  
    - Input: Receives `ai_languageModel`, `ai_tool` (vector retriever), and `ai_memory` (simple memory) inputs  
    - Output: Generates reply text or NO_REPLY_NEEDED  
    - Edge Cases: Logic errors in prompt, unexpected AI responses  

#### 2.7 Reply Handling

- **Overview:**  
Checks if the AI output indicates a reply is needed and creates a Gmail draft reply accordingly.

- **Nodes Involved:**  
  - Check If Reply Needed  
  - Gmail - Create Draft

- **Node Details:**  
  - **Check If Reply Needed**  
    - Type: If Node  
    - Configuration: Condition checks if AI output does NOT contain the exact string `NO_REPLY_NEEDED`  
    - Input: Output of Email Reply Agent  
    - Output: If true, passes data to Gmail Create Draft; else workflow ends or handles differently  
    - Edge Cases: Case sensitivity, partial matches, empty output  
  - **Gmail - Create Draft**  
    - Type: Gmail (draft creation)  
    - Configuration: Creates an HTML draft reply in the same thread, addressed to the original sender, subject prefixed with "Re:"  
    - Input: Output of Check If Reply Needed  
    - Credentials: Gmail OAuth2 required  
    - Edge Cases: Draft creation failures, invalid email addresses, OAuth expiration  

#### 2.8 Error Handling

- **Overview:**  
Catches and logs errors from key nodes to allow debugging and avoid workflow halts.

- **Nodes Involved:**  
  - Error Handler

- **Node Details:**  
  - **Error Handler**  
    - Type: Code Node (JavaScript)  
    - Configuration: Logs error details including timestamp, full error input, workflow name, and threadId if available  
    - Input: Connected as error handler from nodes like Email Reply Agent, Gmail Fetch Thread, Gmail Create Draft, Google Drive Download Profile  
    - Output: Returns JSON with error data for further processing or alerting  
    - Edge Cases: Logging failures, missing error context  

#### 2.9 Workflow Instructions

- **Overview:**  
Provides user-facing documentation and setup instructions directly within the workflow UI as a sticky note.

- **Nodes Involved:**  
  - Workflow Instructions (Sticky Note)

- **Node Details:**  
  - **Workflow Instructions**  
    - Type: Sticky Note  
    - Configuration: Contains setup steps for OAuth2 credentials, file ID update, workflow functionality overview, and feature highlights  
    - Input/Output: None (informational)  

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                   | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                       |
|---------------------------|---------------------------------------------|---------------------------------|--------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Gmail Trigger             | Gmail Trigger                               | Detect incoming emails          | None                           | Google Drive - Download Profile, Gmail - Fetch Thread | "Monitors Gmail for new emails excluding those from self."                                      |
| Google Drive - Download Profile | Google Drive (download)                     | Retrieve user profile document  | Gmail Trigger                  | Profile Document Loader          | "Update YOUR_PROFILE_DOC_FILE_ID with your profile file."                                      |
| Gmail - Fetch Thread      | Gmail (getAll thread messages)              | Fetch full email thread         | Gmail Trigger                  | Thread Document Loader           |                                                                                                 |
| Profile Document Loader   | Langchain Document Loader (JSON)             | Load profile document JSON      | Google Drive - Download Profile | Profile Text Splitter            |                                                                                                 |
| Thread Document Loader    | Langchain Document Loader (JSON)             | Load email thread JSON          | Gmail - Fetch Thread           | Simple Vector Store              |                                                                                                 |
| Profile Text Splitter     | Langchain Recursive Character Text Splitter | Split profile text into chunks  | Profile Document Loader        | OpenAI Embeddings               |                                                                                                 |
| Thread Text Splitter      | Langchain Recursive Character Text Splitter | Split thread text into chunks   | Thread Document Loader         | OpenAI Embeddings               |                                                                                                 |
| OpenAI Embeddings         | Langchain OpenAI Embeddings                   | Generate vector embeddings      | Profile Text Splitter, Thread Text Splitter | Simple Vector Store              | Requires valid OpenAI API credentials                                                           |
| Simple Vector Store       | Langchain Vector Store (In-Memory)            | Store embeddings by thread ID   | OpenAI Embeddings              | Prepare Agent Context            |                                                                                                 |
| Prepare Agent Context     | Set Node                                      | Create AI prompt text           | Simple Vector Store            | Email Reply Agent               |                                                                                                 |
| Vector Store Retriever    | Langchain Vector Store Retriever (tool)       | Retrieve relevant context       | None (uses memory key)         | Email Reply Agent               |                                                                                                 |
| Simple Memory            | Langchain Memory Buffer Window                 | Maintain conversational memory  | None                         | Email Reply Agent               |                                                                                                 |
| OpenAI Chat Model         | Langchain Chat Model Node                       | Generate AI text responses      | None (configured as AI model) | Email Reply Agent               | Requires valid OpenAI API credentials                                                           |
| Email Reply Agent         | Langchain Agent Node                            | Main AI logic for reply         | Prepare Agent Context, Vector Store Retriever, Simple Memory, OpenAI Chat Model | Check If Reply Needed            | System prompt instructs professional, polite, context-aware email responses                     |
| Check If Reply Needed     | If Node                                        | Determine if reply required     | Email Reply Agent             | Gmail - Create Draft            | Checks if output contains "NO_REPLY_NEEDED"                                                     |
| Gmail - Create Draft      | Gmail (draft creation)                          | Create email draft reply        | Check If Reply Needed         | None                          | Creates draft with HTML email, in thread, addressed to original sender                          |
| Error Handler            | Code Node                                      | Log and handle errors           | Multiple nodes (error output)  | Returns error info             | Logs error timestamp, details, workflow name, threadId                                         |
| Workflow Instructions    | Sticky Note                                    | Setup and workflow overview     | None                         | None                          | Contains setup instructions and workflow description                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Parameters:  
     - Poll every minute (`everyMinute`)  
     - Filter query: `-from:me` (exclude self-sent emails)  
   - Credentials: Configure Gmail OAuth2 credentials  
   - Connect: No input; this is the entry trigger  

2. **Create Google Drive - Download Profile Node**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Set to your profile document file ID (`YOUR_PROFILE_DOC_FILE_ID`)  
   - Credentials: Configure Google Drive OAuth2 credentials  
   - Connect: Output of Gmail Trigger → Input of this node  

3. **Create Gmail - Fetch Thread Node**  
   - Type: Gmail  
   - Operation: Get All messages in thread  
   - Thread ID: Expression `={{ $json.threadId }}` from Gmail Trigger  
   - Credentials: Gmail OAuth2 same credential as trigger  
   - Connect: Output of Gmail Trigger → Input of this node  

4. **Create Profile Document Loader Node**  
   - Type: Langchain Document Loader (default JSON mode)  
   - JSON Data: Expression `={{ $json.content }}` (content from Google Drive Download Profile)  
   - Metadata: Add metadata key `type` with value `user_profile`  
   - Connect: Output Google Drive Download Profile → Input this node  

5. **Create Thread Document Loader Node**  
   - Type: Langchain Document Loader (default JSON mode)  
   - JSON Data: Expression `={{ JSON.stringify($json.messages) }}` from Gmail Fetch Thread  
   - Metadata: Add metadata key `type` with value `email_thread`  
   - Connect: Output Gmail Fetch Thread → Input this node  

6. **Create Profile Text Splitter Node**  
   - Type: Langchain Recursive Character Text Splitter  
   - Chunk Size: 1000 characters  
   - Chunk Overlap: 100 characters  
   - Connect: Output Profile Document Loader → Input this node  

7. **Create Thread Text Splitter Node**  
   - Type: Langchain Recursive Character Text Splitter  
   - Chunk Size: 2000 characters  
   - Chunk Overlap: 200 characters  
   - Connect: Output Thread Document Loader → Input this node  

8. **Create OpenAI Embeddings Node**  
   - Type: Langchain OpenAI Embeddings  
   - Credentials: Configure OpenAI API key  
   - Connect: Outputs from both Profile Text Splitter and Thread Text Splitter → Inputs of this node (merge inputs)  

9. **Create Simple Vector Store Node**  
   - Type: Langchain Vector Store In-Memory  
   - Mode: Insert  
   - Memory Key: Use expression `email_context_{{ $('Gmail Trigger').item.json.threadId }}`  
   - Connect: Output OpenAI Embeddings → Input this node  

10. **Create Prepare Agent Context Node**  
    - Type: Set Node  
    - Assignments: Create string variable `text` with multi-line prompt string including:  
      - Incoming email subject, from header, email body as HTML (expressions from Gmail Trigger)  
      - Instructions for AI: no reply to newsletters, generate professional replies, use vector store context  
    - Connect: Output Simple Vector Store → Input this node  

11. **Create Vector Store Retriever Node**  
    - Type: Langchain Vector Store In-Memory  
    - Mode: Retrieve-as-tool  
    - Memory Key: Same as vector store, expression `email_context_{{ $('Gmail Trigger').item.json.threadId }}`  
    - Limit: 5  
    - Tool Name: `email_context`  
    - Tool Description: Retrieves user profile and thread context  
    - Connect: No direct input, used as tool by agent  

12. **Create Simple Memory Node**  
    - Type: Langchain Memory Buffer Window  
    - Session Key: Expression `email_{{ $('Gmail Trigger').item.json.threadId }}`  
    - Max Token Limit: 4000  
    - Session ID Type: Custom Key  
    - Connect: No direct input, used as memory by agent  

13. **Create OpenAI Chat Model Node**  
    - Type: Langchain Chat Model  
    - Model: `gpt-4o-mini`  
    - Temperature: 0.3 (deterministic, professional tone)  
    - Credentials: OpenAI API key  
    - Connect: No direct input, used as AI language model by agent  

14. **Create Email Reply Agent Node**  
    - Type: Langchain Agent  
    - Text Input: Expression `={{ $json.text }}` from Prepare Agent Context  
    - Options System Message: Detailed prompt instructing polite, professional replies referencing profile and thread context, instructions to respond with 'NO_REPLY_NEEDED' for newsletters/spam  
    - Prompt Type: Define  
    - Connect:  
      - Main input from Prepare Agent Context  
      - `ai_languageModel` input from OpenAI Chat Model  
      - `ai_tool` input from Vector Store Retriever  
      - `ai_memory` input from Simple Memory  
    - Connect output main to Check If Reply Needed  
    - Connect output error to Error Handler  

15. **Create Check If Reply Needed Node**  
    - Type: If Node  
    - Condition: Check if AI output text **does not contain** "NO_REPLY_NEEDED"  
    - Connect: Input from Email Reply Agent main output  
    - True output connects to Gmail Create Draft  
    - False output: No action (end)  

16. **Create Gmail - Create Draft Node**  
    - Type: Gmail  
    - Resource: Draft  
    - Email Type: HTML  
    - Subject: Expression `=Re: {{ $('Gmail Trigger').item.json.subject }}`  
    - Send To: Expression `={{ $('Gmail Trigger').item.json.headers.from }}`  
    - Thread ID: Expression `={{ $('Gmail Trigger').item.json.threadId }}`  
    - Message: Expression replacing newlines with `<br />\n` from AI output  
    - Credentials: Gmail OAuth2  
    - Connect: Input from Check If Reply Needed node true output  
    - Error output to Error Handler  

17. **Create Error Handler Node**  
    - Type: Code Node  
    - JavaScript Code: Logs error details (timestamp, error input, workflow name, threadId)  
    - Connect error outputs from: Google Drive Download Profile, Gmail Fetch Thread, Email Reply Agent, Gmail Create Draft → Error Handler  

18. **Add Sticky Note Node**  
    - Type: Sticky Note  
    - Content: Setup instructions for Gmail OAuth2, Google Drive OAuth2, OpenAI API credentials, profile doc file ID update, workflow description and features  
    - Position for user reference only  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow requires valid OAuth2 credentials for Gmail and Google Drive, and a working OpenAI API key. | Credential setup is mandatory before activating the workflow.                                     |
| Update the Google Drive profile document file ID in the respective node before running.              | Replace placeholder `YOUR_PROFILE_DOC_FILE_ID` with actual file ID.                               |
| The AI agent uses GPT-4o-mini model for a balance between performance and cost.                       | Model selection can be changed in the OpenAI Chat Model node if needed.                            |
| The vector store is in-memory and scoped per email thread to provide relevant context for replies.   | For persistent storage, consider integrating a database-backed vector store instead.              |
| The workflow intelligently ignores newsletters, spam, and automated emails by detecting keywords.    | Reply suppression is controlled by the "NO_REPLY_NEEDED" exact output string from the AI agent.   |
| Error handling is centralized to log errors with contextual info such as threadId and timestamp.     | This facilitates troubleshooting without stopping the entire workflow.                            |
| Sticky note within the workflow provides a quick setup and feature overview for users and maintainers.|                                                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---