Create Professional Email Drafts with GPT-4, Telegram & Contact Database

https://n8nworkflows.xyz/workflows/create-professional-email-drafts-with-gpt-4--telegram---contact-database-8395


# Create Professional Email Drafts with GPT-4, Telegram & Contact Database

### 1. Workflow Overview

This workflow automates the creation of professional email drafts by leveraging GPT-4 AI, a Telegram chat interface, and a contact database stored in Pinecone vector search. It is designed to receive user requests via Telegram, enrich these requests with relevant contact details retrieved from a vector database, generate formal email drafts, and send the drafts back to the user with confirmation messages and stickers.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives user messages from Telegram.
- **1.2 AI Processing with Retrieval-Augmented Generation (RAG):** Uses GPT-4 and Pinecone vector stores to generate formal email drafts based on user input and contact data.
- **1.3 Output Parsing and Draft Creation:** Parses AI output, formats it for Gmail draft creation.
- **1.4 Email Draft Creation and User Notification:** Creates Gmail draft emails and sends confirmation messages with stickers back to the user.
- **1.5 Contact Data Loading & Vector Store Insertion:** A disabled manual trigger block to push contact data into Pinecone (used for initialization or updates).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming Telegram messages to trigger the email drafting process.
- **Nodes Involved:** Telegram Trigger
- **Node Details:**
  - **Telegram Trigger**
    - Type: Telegram Trigger node
    - Configuration: Listens to "message" updates only.
    - Credentials: Telegram API OAuth2
    - Input: Telegram chat messages
    - Output: Passes chat message JSON with user text for further processing.
    - Edge Cases: Telegram API connectivity failures, invalid message content.
  
#### 1.2 AI Processing with RAG

- **Overview:** Uses GPT-4 to draft formal emails, with enhanced context from contact data retrieved from Pinecone vector index.
- **Nodes Involved:** AI Agent, Pinecone Vector Store1, OpenAI Chat Model, Embeddings OpenAI1
- **Node Details:**
  - **AI Agent**
    - Type: LangChain Agent node
    - Role: Main AI processor that receives Telegram message text, calls Pinecone to retrieve contact emails, and generates a formal email draft in strict JSON format.
    - Configuration:
      - Input text: Telegram message text expression `{{$json.message.text}}`
      - System prompt directs the AI to always produce formal emails, use Pinecone to fetch recipient emails, and format output as JSON with keys like `sendTo`, `subject`, `message`, etc.
      - Sends queries to Pinecone Vector Store1 as a tool for contact data retrieval.
      - Fixed sender name "Abbas Alaa".
    - Input: Telegram message text
    - Output: JSON-formatted email draft query string.
    - Edge Cases: Pinecone retrieval failure, JSON parsing errors, AI output formatting errors.
    - Version-specific: LangChain Agent v2.2.
  
  - **Pinecone Vector Store1**
    - Type: LangChain Pinecone vector store node (retrieve-as-tool)
    - Role: Provides contact email lookup for AI Agent tool use.
    - Configuration:
      - Mode: Retrieve-as-tool
      - Namespace: "contacts"
      - Index: "gmailagent"
      - Tool Description: Fetches contact information such as email addresses.
    - Credentials: Pinecone API
    - Input: Embeddings from OpenAI and AI Agent queries
    - Output: Contact data results for AI Agent
    - Edge Cases: Pinecone API key errors, network issues, empty retrieval.
  
  - **OpenAI Chat Model**
    - Type: LangChain Chat model node
    - Role: Provides GPT-4 powered language model for the AI Agent.
    - Configuration:
      - Model: "gpt-4.1-mini"
      - No extra options set.
    - Credentials: OpenAI API key
    - Input: Chat messages from AI Agent
    - Output: Draft email JSON string
    - Edge Cases: OpenAI API rate limits, timeouts.
  
  - **Embeddings OpenAI1**
    - Type: LangChain OpenAI embeddings node
    - Role: Provides vector embeddings for Pinecone retrieval.
    - Configuration:
      - Dimensions: 512
    - Credentials: OpenAI API
    - Input: Text from AI Agent for embedding
    - Output: Embeddings sent to Pinecone Vector Store1
    - Edge Cases: API limits.
  
#### 1.3 Output Parsing and Draft Creation

- **Overview:** Parses the AI-generated JSON draft string, extracts email parameters, and prepares data for Gmail draft creation.
- **Nodes Involved:** Code, Create a draft
- **Node Details:**
  - **Code**
    - Type: Code node (JavaScript)
    - Role: Cleans AI output by removing markdown code fences, parsing JSON, decoding URL-encoded query parameters, and extracting fields like `sendTo`, `subject`, `message`, etc.
    - Configuration: Custom JS code to validate and transform AI output.
    - Input: AI Agent JSON output
    - Output: Structured JSON with email draft fields for Gmail node.
    - Edge Cases: JSON parse errors, missing keys, malformed AI output.
  
  - **Create a draft**
    - Type: Gmail node
    - Role: Creates a draft email in Gmail using parsed parameters.
    - Configuration:
      - Resource: draft
      - Uses fields from Code node: sendTo, subject, message, ccList, bccList.
    - Credentials: Gmail OAuth2
    - Input: Parsed email parameters
    - Output: Gmail draft creation result
    - Edge Cases: Gmail API auth errors, draft creation failure.

#### 1.4 Email Draft Creation and User Notification

- **Overview:** Sends confirmation text and a sticker back to the Telegram user after the draft creation.
- **Nodes Involved:** Send a text message, Send a sticker
- **Node Details:**
  - **Send a text message**
    - Type: Telegram node (send message)
    - Role: Sends a confirmation message ("تم تآمر آمر") to the Telegram user.
    - Configuration: Text message set in Arabic, dynamically uses chatId from Telegram Trigger.
    - Credentials: Telegram API
    - Input: Output from Create a draft node
    - Output: Confirmation message sent
    - Edge Cases: Telegram API failures, invalid chat ID.
  
  - **Send a sticker**
    - Type: Telegram node (send sticker)
    - Role: Sends a funny sticker to the user as a completion acknowledgement.
    - Configuration:
      - Sticker file ID provided
      - Chat ID from Telegram Trigger
    - Credentials: Telegram API
    - Input: After Send a text message
    - Output: Sticker sent
    - Edge Cases: Sticker file ID invalid, Telegram API errors.

#### 1.5 Contact Data Loading & Vector Store Insertion (Disabled Manual Trigger)

- **Overview:** A manual trigger node to initiate pushing contact data into the Pinecone vector store.
- **Nodes Involved:** Manual Trigger ("When clicking 'Execute workflow'"), Get a document, Default Data Loader, Embeddings OpenAI, Pinecone Vector Store
- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger (disabled)
    - Role: Allows user to trigger contact data ingestion manually.
  
  - **Get a document**
    - Type: Google Docs node
    - Role: Retrieves a Google Docs document containing contact data.
    - Configuration: Document URL specified.
    - Credentials: Google Docs OAuth2
    - Output: Document content to be processed.
  
  - **Default Data Loader**
    - Type: Document loader node
    - Role: Processes document content for vectorization.
  
  - **Embeddings OpenAI**
    - Type: Embeddings node
    - Role: Creates embeddings from document content for Pinecone indexing.
  
  - **Pinecone Vector Store**
    - Type: Pinecone vector insertion node
    - Role: Inserts embeddings into Pinecone index "gmailagent" under namespace "contacts".
    - Credentials: Pinecone API
  - Edge Cases: Google Docs access issues, Pinecone insertion errors.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                                | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                 |
|----------------------------|---------------------------------------|-----------------------------------------------|----------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                        | Manual start to push contact data to Pinecone | None                       | Get a document             | Send data to vector database. Connect trigger node when ready to push contacts data to the database.         |
| Get a document             | Google Docs                           | Retrieve contact data document                 | When clicking ‘Execute workflow’ | Pinecone Vector Store       |                                                                                                             |
| Default Data Loader        | Document Loader                      | Prepare document content for embedding         | Get a document             | Pinecone Vector Store       |                                                                                                             |
| Embeddings OpenAI          | OpenAI Embeddings                    | Generate text embeddings for Pinecone          | Default Data Loader        | Pinecone Vector Store       |                                                                                                             |
| Pinecone Vector Store      | Pinecone Vector Store (Insert)       | Insert contact embeddings into Pinecone index  | Embeddings OpenAI          |                            |                                                                                                             |
| Telegram Trigger           | Telegram Trigger                      | Receive user messages from Telegram             | None                       | AI Agent                   |                                                                                                             |
| AI Agent                  | LangChain Agent                       | Generate formal email drafts using GPT-4 and contact retrieval | Telegram Trigger           | Code                       | RAG AI agent to get the "send to" emails and format the emails                                              |
| OpenAI Chat Model          | OpenAI Chat Model                    | Provide GPT-4 language model for AI Agent      | AI Agent                   | AI Agent                   |                                                                                                             |
| Embeddings OpenAI1         | OpenAI Embeddings                    | Generate embeddings for Pinecone retrieval tool | AI Agent                   | Pinecone Vector Store1      |                                                                                                             |
| Pinecone Vector Store1     | Pinecone Vector Store (Retrieve)     | Retrieve contact info for AI Agent tool         | Embeddings OpenAI1         | AI Agent                   |                                                                                                             |
| Code                      | Code                                 | Parse AI JSON output to extract email fields   | AI Agent                   | Create a draft             |                                                                                                             |
| Create a draft            | Gmail                                | Create Gmail draft email using parsed data      | Code                       | Send a text message        | Draft the email and send the completion message with a funny sticker.                                       |
| Send a text message        | Telegram                             | Send confirmation text message to user          | Create a draft             | Send a sticker             |                                                                                                             |
| Send a sticker            | Telegram                             | Send funny sticker to user as confirmation      | Send a text message        |                            |                                                                                                             |
| Sticky Note               | Sticky Note                         | Instructional note for manual trigger block     | None                       | None                       | Send data to vector database. Connect trigger node when ready to push your contacts data to the database.    |
| Sticky Note1              | Sticky Note                         | Instructional note for RAG AI agent block       | None                       | None                       | RAG AI agent to get the "send to" emails and format the emails                                              |
| Sticky Note2              | Sticky Note                         | Instructional note for email draft and notification | None                       | None                       | Draft the email and send the completion message with a funny sticker.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - Disabled by default  
   - Purpose: To manually trigger contact data ingestion.

2. **Add Google Docs Node**  
   - Type: Google Docs  
   - Name: "Get a document"  
   - Operation: Get  
   - Document URL: Set to your Google Docs containing contacts data  
   - Credentials: Google Docs OAuth2

3. **Add Document Default Data Loader Node**  
   - Type: LangChain Document Default Data Loader  
   - Name: "Default Data Loader"  
   - No special options

4. **Add OpenAI Embeddings Node**  
   - Type: LangChain Embeddings OpenAI  
   - Name: "Embeddings OpenAI"  
   - Dimensions: 512  
   - Credentials: OpenAI API

5. **Add Pinecone Vector Store Node (Insert Mode)**  
   - Type: LangChain Vector Store Pinecone  
   - Name: "Pinecone Vector Store"  
   - Mode: Insert  
   - Pinecone Namespace: "contacts"  
   - Pinecone Index: "gmailagent"  
   - Credentials: Pinecone API

6. **Connect the above nodes in order:**  
   Manual Trigger → Get a document → Default Data Loader → Embeddings OpenAI → Pinecone Vector Store

7. **Add Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Name: "Telegram Trigger"  
   - Updates: message  
   - Credentials: Telegram API OAuth2

8. **Add LangChain AI Agent Node**  
   - Type: LangChain Agent  
   - Name: "AI Agent"  
   - Text Input: `={{ $json.message.text }}` from Telegram Trigger  
   - Prompt: Include system message instructing to draft formal emails, always use Pinecone tool to retrieve recipient email, output strict JSON, sender name fixed as "Abbas Alaa"  
   - Tools: Add Pinecone Vector Store as a tool (see next steps)  
   - Credentials: OpenAI API

9. **Add LangChain Pinecone Vector Store Node (Retrieve-as-tool)**  
   - Type: LangChain Vector Store Pinecone  
   - Name: "Pinecone Vector Store1"  
   - Mode: Retrieve-as-tool  
   - Pinecone Namespace: "contacts"  
   - Pinecone Index: "gmailagent"  
   - Tool Description: "Call this to retrieve information about contacts like email address."  
   - Credentials: Pinecone API

10. **Add OpenAI Chat Model Node**  
    - Type: LangChain LM Chat OpenAI  
    - Name: "OpenAI Chat Model"  
    - Model: gpt-4.1-mini  
    - Credentials: OpenAI API

11. **Add OpenAI Embeddings Node for Retrieval**  
    - Type: LangChain Embeddings OpenAI  
    - Name: "Embeddings OpenAI1"  
    - Dimensions: 512  
    - Credentials: OpenAI API

12. **Connect AI Agent inputs and outputs:**  
    - Telegram Trigger → AI Agent (text input)  
    - AI Agent → OpenAI Chat Model (ai_languageModel)  
    - AI Agent → Pinecone Vector Store1 (ai_tool)  
    - AI Agent → Embeddings OpenAI1 (ai_embedding)  
    - Embeddings OpenAI1 → Pinecone Vector Store1 (ai_embedding)  

13. **Add Code Node to Parse AI Output**  
    - Type: Code  
    - Name: "Code"  
    - JavaScript code to:  
      - Remove markdown code fences from AI output  
      - Parse JSON string  
      - Extract query string  
      - Split key-value pairs and decode values  
      - Return JSON object with sendTo, subject, message, ccList, bccList, senderName, emailType  
    - Input: AI Agent output

14. **Add Gmail Node to Create Draft**  
    - Type: Gmail  
    - Name: "Create a draft"  
    - Resource: Draft  
    - Parameters:  
      - sendTo: `={{ $json.sendTo }}`  
      - subject: `={{ $json.subject }}`  
      - message: `={{ $json.message }}`  
      - ccList: `={{ $json.ccList }}`  
      - bccList: `={{ $json.bccList }}`  
    - Credentials: Gmail OAuth2

15. **Add Telegram Node to Send Confirmation Text**  
    - Type: Telegram  
    - Name: "Send a text message"  
    - Text: "تم تآمر آمر"  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Credentials: Telegram API OAuth2

16. **Add Telegram Node to Send Sticker**  
    - Type: Telegram  
    - Name: "Send a sticker"  
    - File: Sticker ID `CAACAgIAAxkBAANEaL7Y0gABEcXooA-abKgPERguErxVAAIHRwAC839wS2T25-Ght_GiNgQ`  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Credentials: Telegram API OAuth2

17. **Connect nodes for output flow:**  
    AI Agent → Code → Create a draft → Send a text message → Send a sticker

18. **Optional:** Add Sticky Notes for instructions on pushing contacts data and AI agent functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The sender name is always fixed as "Abbas Alaa" in the AI prompt to ensure consistency in outgoing emails.    | AI Agent system message                                                                             |
| The AI output must strictly be JSON formatted with no markdown or extra text for reliable parsing.             | AI Agent instructions and Code node parsing logic                                                  |
| Pinecone namespace "contacts" and index "gmailagent" are central for contact email retrieval and insertion.    | Pinecone Vector Store nodes                                                                        |
| Telegram messages trigger the entire email drafting and sending process, enabling conversational email creation.| Telegram Trigger node                                                                               |
| Gmail drafts are created but not sent automatically; user can review before sending.                           | Gmail "Create a draft" node                                                                        |
| Confirmation messages and stickers are sent to Telegram users to improve UX with friendly feedback.            | Telegram Send a text message and Send a sticker nodes                                             |
| Google Docs document URL must be accessible with proper OAuth2 credentials for contact data import.            | Get a document node setup                                                                          |
| OpenAI GPT-4 model selected is "gpt-4.1-mini" – ensure API access and quotas are sufficient.                   | OpenAI Chat Model node                                                                             |
| For more information on Pinecone and LangChain integration, consult: <https://www.pinecone.io/docs/>           | Pinecone official documentation                                                                    |
| For n8n LangChain nodes usage and configuration, see: <https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/> | n8n LangChain integration docs                                                                     |

---

This detailed reference document allows advanced users and automation agents to understand, reproduce, and maintain the "Create Professional Email Drafts with GPT-4, Telegram & Contact Database" workflow fully and reliably.