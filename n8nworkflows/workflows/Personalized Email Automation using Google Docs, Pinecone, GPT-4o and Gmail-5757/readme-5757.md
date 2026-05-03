Personalized Email Automation using Google Docs, Pinecone, GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/personalized-email-automation-using-google-docs--pinecone--gpt-4o-and-gmail-5757


# Personalized Email Automation using Google Docs, Pinecone, GPT-4o and Gmail

### 1. Workflow Overview

This workflow entitled **"Personalized Email Automation using Google Docs, Pinecone, GPT-4o and Gmail"** automates the process of generating and sending personalized emails leveraging AI and vector search technologies. The core use case is enabling dynamic, context-aware email communication based on contact data stored in Google Docs and indexed in Pinecone, enhanced by GPT-4o AI agents that compose and dispatch emails individually via Gmail.

Logical blocks are organized as follows:

- **1.1 Data Ingestion & Vectorization:** Reads contact email data from a Google Docs document, processes it into vector embeddings using OpenAI Embeddings, and inserts these vectors into a Pinecone vector database for fast semantic retrieval.

- **1.2 AI-Powered Email Query & Composition:** Triggered by incoming chat messages, an AI Agent uses tools connected to the Pinecone vector store to retrieve relevant email addresses and another tool to send emails. This agent dynamically generates personalized email content using GPT-4o, ensuring professional formatting and personalization.

- **1.3 Email Sending Execution:** A subsidiary workflow receives the generated email data, uses another AI Agent to finalize message formatting using GPT-4o-mini, and sends the actual email through Gmail.

Each block corresponds to a phase in the email automation pipeline, incorporating data retrieval, AI-driven composition, and delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Vectorization

- **Overview:**  
  This block reads a Google Docs document containing contact emails, splits and processes the text into chunks, generates vector embeddings, and inserts/upserts them into a Pinecone vector store to enable semantic search.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get a document (Google Docs)  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Pinecone Vector Store

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow manually for testing or ad-hoc runs.  
    - *Connections:* Output to *Get a document*.  
    - *Failure Modes:* None intrinsic; user-initiated.

  - **Get a document**  
    - *Type:* Google Docs node  
    - *Role:* Retrieves the content of a specified Google Docs document by URL.  
    - *Configuration:* Operation = "get", Document URL specified (redacted here).  
    - *Input:* Trigger from manual node.  
    - *Output:* Full text content to *Pinecone Vector Store* node.  
    - *Failure Modes:* API authorization errors, document access permission issues, invalid URL.

  - **Recursive Character Text Splitter**  
    - *Type:* Text Splitter (Recursive Character)  
    - *Role:* Splits long document text into smaller chunks for better embedding generation.  
    - *Configuration:* chunkSize=200 characters, chunkOverlap=50 characters.  
    - *Input:* Text from *Default Data Loader*.  
    - *Output:* Chunks passed to *Embeddings OpenAI*.  
    - *Failure Modes:* Improper splitting if text is too short or malformed.

  - **Default Data Loader**  
    - *Type:* Document Data Loader  
    - *Role:* Prepares document text for processing by the splitter and embedding nodes.  
    - *Input:* Output from Text Splitter.  
    - *Output:* Document chunks to *Pinecone Vector Store*.  
    - *Failure Modes:* Data formatting errors.

  - **Embeddings OpenAI**  
    - *Type:* OpenAI Embeddings Node  
    - *Role:* Converts text chunks into vector embeddings using OpenAI's embedding model.  
    - *Configuration:* Default options; no custom parameters.  
    - *Input:* Text chunks from *Recursive Character Text Splitter*.  
    - *Output:* Embeddings to *Pinecone Vector Store*.  
    - *Failure Modes:* API quota limits, network errors, malformed inputs.

  - **Pinecone Vector Store**  
    - *Type:* Pinecone Vector Store Node  
    - *Role:* Inserts embedding vectors into the Pinecone index "n8ndocs" under namespace "docsmail".  
    - *Configuration:* Mode = "insert", Pinecone index and namespace specified.  
    - *Input:* Embeddings from *Embeddings OpenAI*.  
    - *Output:* Final node in ingestion chain.  
    - *Failure Modes:* Pinecone API errors, authentication failures, vector dimensionality mismatch.

- **Sticky Note Context:**  
  Describes the first step of reading Google Docs data, embedding, and inserting into Pinecone.

---

#### 2.2 AI-Powered Email Query & Composition

- **Overview:**  
  This block listens for incoming chat messages, uses an AI Agent configured with GPT-4o to interpret queries, retrieve relevant emails from Pinecone, and invoke a separate workflow to send personalized emails.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - AI Agent1 (Langchain Agent with detailed system message)  
  - Vectorstore Mails (Tool Vector Store Node)  
  - Pinecone Vector Store1  
  - Embeddings OpenAI1  
  - OpenAI Chat Model1  
  - send_mail (Tool Workflow node)  
  - OpenAI Chat Model2

- **Node Details:**

  - **When chat message received**  
    - *Type:* Langchain Chat Trigger  
    - *Role:* Listens for incoming chat messages to initiate AI-driven email processing.  
    - *Input:* External chat webhook trigger.  
    - *Output:* Triggers *AI Agent1*.  
    - *Failure Modes:* Webhook connectivity, message parsing errors.

  - **AI Agent1**  
    - *Type:* Langchain Agent with AI Tools  
    - *Role:* Core AI logic that receives chat queries, uses two tools: Vectorstore_mails to find emails and send_mail to dispatch emails.  
    - *Configuration:* Custom system message defining agent role and usage of tools, emphasizing email personalization and format rules. Uses GPT-4o.  
    - *Input:* Query text from chat trigger.  
    - *Output:* Calls tools *Vectorstore Mails* and *send_mail*.  
    - *Failure Modes:* AI model errors, tool invocation failures, malformed outputs.

  - **Vectorstore Mails**  
    - *Type:* Tool Vector Store (Langchain)  
    - *Role:* Enables AI Agent to query Pinecone vector store for email addresses using semantic search.  
    - *Configuration:* Linked with Pinecone Vector Store1 and Embeddings OpenAI1 nodes. The description guides its usage for email retrieval.  
    - *Input:* Queries from *AI Agent1*.  
    - *Output:* Email addresses back to *AI Agent1*.  
    - *Failure Modes:* Pinecone or embedding service downtime, no matching vectors found.

  - **Pinecone Vector Store1**  
    - *Type:* Pinecone Vector Store  
    - *Role:* Provides vector database access for the Vectorstore Mails tool. Uses same index "n8ndocs" and namespace "docsmail".  
    - *Input:* Embeddings from *Embeddings OpenAI1*.  
    - *Output:* Vectors for retrieval queries.  
    - *Failure Modes:* Same as Pinecone Vector Store node.

  - **Embeddings OpenAI1**  
    - *Type:* OpenAI Embeddings  
    - *Role:* Provides embedding generation for Vectorstore Mails tool queries.  
    - *Input:* Text chunks or queries from AI Agent tools.  
    - *Output:* Embeddings to Pinecone Vector Store1.  
    - *Failure Modes:* API errors.

  - **OpenAI Chat Model1**  
    - *Type:* OpenAI Chat Model (Langchain)  
    - *Role:* Language model supporting Vectorstore Mails tool for semantic understanding and query processing. Uses GPT-4o.  
    - *Input:* Prompts from *Vectorstore Mails*.  
    - *Output:* Processed results to Vectorstore Mails.  
    - *Failure Modes:* API errors, rate limits.

  - **send_mail**  
    - *Type:* Tool Workflow Node  
    - *Role:* Invokes a separate workflow named "Send Mails Pinecone" to handle the actual sending of emails with dynamically generated content.  
    - *Configuration:* References external workflow ID "GLkrSkyYtlep3P6e" with defined inputs; no input mapping defined here (empty).  
    - *Input:* Triggered by *AI Agent1* after email content generation.  
    - *Output:* None in this workflow (sub-workflow handles sending).  
    - *Failure Modes:* Workflow invocation errors, parameter mismatches.

  - **OpenAI Chat Model2**  
    - *Type:* OpenAI Chat Model  
    - *Role:* Supports AI Agent1 with language modeling in email composition. Uses GPT-4o.  
    - *Input:* From AI Agent1.  
    - *Output:* To AI Agent1.  
    - *Failure Modes:* API connectivity, quota.

- **Sticky Note Context:**  
  Describes the second step: AI Agent uses Vectorstore_mails and send_mail tools to retrieve emails and send personalized messages, triggered by chat input.

---

#### 2.3 Email Sending Execution

- **Overview:**  
  This block is triggered as a sub-workflow by the previous block to finalize email content using a smaller GPT-4o-mini model and send the email via Gmail.

- **Nodes Involved:**  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - AI Agent (Langchain Agent)  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Gmail

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point triggered by the parent workflow to receive email details (To, Subject, Message).  
    - *Input:* Passes all inputs directly to next node.  
    - *Output:* To *AI Agent*.  
    - *Failure Modes:* Trigger connectivity, input data errors.

  - **AI Agent**  
    - *Type:* Langchain Agent (GPT-4o-mini)  
    - *Role:* Processes incoming email data, formats and prepares the To, Subject, and Message fields for sending. Acts as a helpful assistant.  
    - *Configuration:* System message specifies role as helpful assistant to send mails; prompt text is the query received.  
    - *Input:* From trigger node.  
    - *Output:* To *Gmail* node with email components.  
    - *Failure Modes:* AI processing errors, malformed input data.

  - **OpenAI Chat Model**  
    - *Type:* OpenAI Chat Model (GPT-4o-mini)  
    - *Role:* Language model used by AI Agent for composing and refining email content.  
    - *Input:* Prompt from AI Agent.  
    - *Output:* To AI Agent.  
    - *Failure Modes:* API downtime, rate limits.

  - **Gmail**  
    - *Type:* Gmail Node  
    - *Role:* Sends the finalized email using Gmail SMTP or API with To, Subject, and Message extracted from AI Agent outputs.  
    - *Configuration:* Uses OAuth2 credentials for Gmail; dynamic fields populated by AI outputs.  
    - *Input:* Email details from AI Agent.  
    - *Output:* Sends email.  
    - *Failure Modes:* Authentication errors, sending limits, invalid email formats.

- **Sticky Note Context:**  
  Details the third step: receiving email details from parent workflow, using AI to format, and sending via Gmail.

---

### 3. Summary Table

| Node Name                    | Node Type                                      | Functional Role                        | Input Node(s)                 | Output Node(s)                | Sticky Note                                       |
|------------------------------|------------------------------------------------|-------------------------------------|------------------------------|------------------------------|--------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                                  | Manual start of workflow             |                              | Get a document                | First Step: Add contacts from Google Docs to Pinecone |
| Get a document                | Google Docs                                     | Retrieve contacts document           | When clicking ‘Test workflow’| Pinecone Vector Store         | First Step: Read Google Docs, process emails      |
| Recursive Character Text Splitter | Text Splitter (Recursive Character)          | Split document text for embedding    | Default Data Loader           | Embeddings OpenAI            | First Step: Prepare chunks for embeddings         |
| Default Data Loader           | Document Data Loader                            | Prepare document text                 | Recursive Character Text Splitter | Pinecone Vector Store     | First Step: Data preparation for Pinecone         |
| Embeddings OpenAI            | OpenAI Embeddings                               | Generate vector embeddings            | Recursive Character Text Splitter | Pinecone Vector Store      | First Step: Generate embeddings for Pinecone      |
| Pinecone Vector Store         | Pinecone Vector Store                           | Insert embeddings into Pinecone       | Embeddings OpenAI, Default Data Loader, Get a document | — | First Step: Upsert vectors to Pinecone            |
| When chat message received    | Langchain Chat Trigger                          | Trigger on chat input                 |                              | AI Agent1                    | Second Step: Trigger AI email query and composition |
| AI Agent1                    | Langchain Agent                                 | Core AI agent for email query & send | When chat message received    | Vectorstore Mails, send_mail | Second Step: Uses tools Vectorstore_mails and send_mail |
| Vectorstore Mails             | Tool Vector Store                               | Retrieve emails from Pinecone         | AI Agent1                    | AI Agent1                   | Second Step: Query Pinecone for emails             |
| Pinecone Vector Store1        | Pinecone Vector Store                           | Vector DB for email retrieval         | Embeddings OpenAI1           | Vectorstore Mails           | Second Step: Pinecone index access                  |
| Embeddings OpenAI1           | OpenAI Embeddings                               | Generate embeddings for retrieval     | —                            | Pinecone Vector Store1      | Second Step: Embeddings for Vectorstore_mails      |
| OpenAI Chat Model1            | OpenAI Chat Model (GPT-4o)                      | Language model for Vectorstore Mails  | Vectorstore Mails            | Vectorstore Mails           | Second Step: Supports semantic search               |
| send_mail                    | Tool Workflow                                   | Send email via external workflow      | AI Agent1                   | —                           | Second Step: Calls "Send Mails Pinecone" workflow  |
| OpenAI Chat Model2            | OpenAI Chat Model (GPT-4o)                      | Language model for AI Agent1          | AI Agent1                   | AI Agent1                   | Second Step: Email generation support                |
| When Executed by Another Workflow | Execute Workflow Trigger                      | Triggered by parent workflow          |                              | AI Agent                    | Third Step: Entry point for email sending workflow  |
| AI Agent                     | Langchain Agent (GPT-4o-mini)                    | Format email content for sending      | When Executed by Another Workflow | Gmail                    | Third Step: Compose and finalize email content      |
| OpenAI Chat Model             | OpenAI Chat Model (GPT-4o-mini)                  | Language model for email formatting   | AI Agent                    | AI Agent                    | Third Step: Support AI Agent email composition      |
| Gmail                        | Gmail Node                                      | Send email via Gmail                   | AI Agent                    | —                           | Third Step: Send email via Gmail                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually for testing.

2. **Add Google Docs Node**  
   - Type: Google Docs  
   - Operation: Get document  
   - Document URL: Set your Google Docs URL containing contact emails.  
   - Connect Manual Trigger output to this node.

3. **Add Recursive Character Text Splitter Node**  
   - Type: Text Splitter (Recursive Character)  
   - Chunk Size: 200 chars  
   - Chunk Overlap: 50 chars  
   - Connect output from Google Docs node.

4. **Add Default Data Loader Node**  
   - Type: Default Data Loader (Langchain Document)  
   - Connect output from Recursive Character Text Splitter.

5. **Add Embeddings OpenAI Node**  
   - Type: OpenAI Embeddings  
   - Use default settings.  
   - Connect output from Recursive Character Text Splitter.

6. **Add Pinecone Vector Store Node**  
   - Type: Pinecone Vector Store  
   - Mode: Insert  
   - Pinecone Index: "n8ndocs" (must exist in your Pinecone account)  
   - Namespace: "docsmail"  
   - Connect embeddings output from Embeddings OpenAI node and document data from Default Data Loader.  
   - Connect also the Google Docs node output (as per the original input chaining).

7. **Add Langchain Chat Trigger Node**  
   - Type: Chat Trigger  
   - Set webhook if necessary.  
   - This will listen for chat requests to start email composition.

8. **Add AI Agent Node (AI Agent1)**  
   - Type: Langchain Agent with tools  
   - Configure system message with detailed instructions for email generation and tool usage (see original system message).  
   - Model: GPT-4o  
   - Tools:  
     - Tool 1: Vectorstore Mails (linked to Pinecone Vector Store1)  
     - Tool 2: send_mail (linked to external workflow)  
   - Connect chat trigger output to this AI Agent.

9. **Add Vectorstore Mails Tool Node**  
   - Type: Tool Vector Store  
   - Description: For retrieving email addresses from Pinecone.  
   - Connect to: Pinecone Vector Store1 and Embeddings OpenAI1 nodes.

10. **Add Pinecone Vector Store1 Node**  
    - Type: Pinecone Vector Store  
    - Same index and namespace as initial Pinecone node.  
    - Connect input from Embeddings OpenAI1.

11. **Add Embeddings OpenAI1 Node**  
    - Type: OpenAI Embeddings  
    - Default configuration.  
    - Connect input as needed to generate embeddings for Vectorstore Mails queries.

12. **Add OpenAI Chat Model1 Node**  
    - Type: OpenAI Chat Model (GPT-4o)  
    - Connect to Vectorstore Mails for semantic query support.

13. **Add Tool Workflow Node (send_mail)**  
    - Type: Tool Workflow  
    - Workflow ID: Link your separate "Send Mails Pinecone" workflow.  
    - Used by AI Agent1 to send emails.

14. **Add OpenAI Chat Model2 Node**  
    - Type: OpenAI Chat Model (GPT-4o)  
    - Connect to AI Agent1 for email generation support.

15. **Add Execute Workflow Trigger Node**  
    - Type: Execute Workflow Trigger  
    - This node allows the workflow to be triggered by the external workflow (send_mail).  
    - Input set to passthrough.

16. **Add AI Agent Node (for email finalization)**  
    - Type: Langchain Agent  
    - Model: GPT-4o-mini  
    - System message: "You are a helpful assistant to send mails."  
    - Input: passthrough from Execute Workflow Trigger.  
    - Output: To Gmail node.

17. **Add OpenAI Chat Model Node**  
    - Type: OpenAI Chat Model (GPT-4o-mini)  
    - Connect input/output to AI Agent node.

18. **Add Gmail Node**  
    - Type: Gmail  
    - Configure OAuth2 credentials for Gmail access.  
    - Parameters:  
      - sendTo: dynamic from AI Agent output "To" field.  
      - subject: dynamic from AI Agent output "Subject" field.  
      - message: dynamic from AI Agent output "Message" field.  
    - Connect output from AI Agent.

19. **Connect nodes appropriately following the described flow.**

20. **Test the entire flow starting from manual trigger or chat message.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| First Step: Read contact emails from Google Docs, generate embeddings with OpenAI, and upsert to Pinecone vector store.                                                                                                          | Sticky Note at workflow start                                                                                                                                         |
| Second Step: AI Agent uses GPT-4o and two tools (Vectorstore_mails for email retrieval, send_mail for sending) triggered by chat messages to generate personalized emails.                                                        | Sticky Note near AI Agent1 and Vectorstore Mails                                                                                                                     |
| Third Step: A sub-workflow triggered by the parent passes email data to another AI Agent (GPT-4o-mini) to finalize email content and send it through Gmail.                                                                      | Sticky Note near Execute Workflow Trigger and Gmail nodes                                                                                                            |
| System messages guide AI Agents for professional, personalized email generation including dynamic placeholders and formatting rules.                                                                                            | Embedded in AI Agent1 node configuration                                                                                                                             |
| Workflow integrates multiple n8n Langchain nodes requiring OpenAI API key and Pinecone credentials to be configured in n8n credentials securely.                                                                                | General integration requirement                                                                                                                                      |
| Pinecone index "n8ndocs" and namespace "docsmail" must be pre-created and accessible.                                                                                                                                              | Pinecone setup prerequisite                                                                                                                                          |
| Gmail node requires OAuth2 authentication with appropriate scopes for sending emails on behalf of the user.                                                                                                                      | Gmail node credential setup                                                                                                                                           |
| The external workflow "Send Mails Pinecone" must exist and be accessible by its ID; it handles the actual email sending process.                                                                                                | External workflow dependency                                                                                                                                          |
| See n8n documentation for Langchain nodes and Pinecone integration for troubleshooting any API or data processing errors.                                                                                                        | https://docs.n8n.io/integrations/builtin/n8n-nodes/langchain/                                                                                                        |

---

**Disclaimer:** The text provided is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.