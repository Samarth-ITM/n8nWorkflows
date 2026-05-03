Automated Product Email Support with GPT-4.1, Pinecone RAG, and Gmail

https://n8nworkflows.xyz/workflows/automated-product-email-support-with-gpt-4-1--pinecone-rag--and-gmail-10194


# Automated Product Email Support with GPT-4.1, Pinecone RAG, and Gmail

### 1. Workflow Overview

This workflow automates email support for a men's clothing store by integrating Gmail, OpenAI’s GPT-4.1 language model, and Pinecone vector store for Retrieval-Augmented Generation (RAG). It processes incoming emails, assesses whether a reply is needed, and if so, generates a tailored, context-aware, and empathetic response based on a custom knowledge base. The workflow logically divides into four functional blocks:

- **1.1 Email Collection & Trigger:** Monitor incoming Gmail messages excluding self-sent emails.
- **1.2 Reply Necessity Assessment:** Use GPT-4.1 to analyze if the email requires a response, filtering only relevant queries about men’s clothing.
- **1.3 AI Response Generation with RAG:** If a reply is required, employ a RAG agent that combines Pinecone-powered vector search with GPT-4.1 to compose a professional email reply.
- **1.4 Sending the Reply:** Use Gmail node to send the generated answer as a direct reply to the original email.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Collection & Trigger

- **Overview:**  
  This block listens for new incoming emails in the Gmail account, filtering out emails sent by the account owner, thus only capturing customer inquiries or external messages.

- **Nodes Involved:**  
  - Gmail Trigger1  
  - Sticky Note (Collecting Received Email)

- **Node Details:**

  - **Gmail Trigger1**  
    - Type: Gmail Trigger  
    - Role: Watches inbox for new incoming emails every minute, filtering out emails sent from self (`q: -from:me`).  
    - Configuration: Polling every minute; simple mode disabled for advanced filters.  
    - Inputs: None (trigger node)  
    - Outputs: Email JSON data including subject, headers, and message content  
    - Potential Failures: OAuth2 token expiry or Gmail API rate limits.

  - **Sticky Note (Collecting Received Email)**  
    - Type: Sticky Note  
    - Purpose: Visual annotation marking the email collection phase.

#### 2.2 Reply Necessity Assessment

- **Overview:**  
  Uses OpenAI GPT-4.1 to determine if an incoming email requires a reply. It filters out irrelevant emails, especially those not related to the men's clothing store, outputting a boolean flag.

- **Nodes Involved:**  
  - Assess if message needs a reply1  
  - OpenAI Chat1  
  - JSON Parser1  
  - If Needs Reply1  
  - Sticky Note1 (Assessing whether mail needs reply)  
  - Sticky Note2 (Filtering mails needing reply)

- **Node Details:**

  - **Assess if message needs a reply1**  
    - Type: Chain LLM (Language Model Chain)  
    - Role: Forms prompt with email subject and HTML text, instructs GPT-4.1 to output JSON `{ "needsReply": true/false }` based on relevance to men’s clothing store.  
    - Key Expression:  
      - Prompt Template:  
        ```
        Subject: {{ $json.subject }}
        Message:
        {{ $json.textAsHtml }}
        ```
      - Instruction: Return JSON with `needsReply` boolean.  
    - Input: Output of Gmail Trigger1  
    - Output: JSON object with `needsReply` property  
    - Failure Modes: Model response parsing errors, API timeouts.

  - **OpenAI Chat1**  
    - Type: OpenAI Chat Model  
    - Role: Executes the GPT-4.1 model for the assessment prompt.  
    - Configuration: Temperature 0 for deterministic output, response format JSON object.  
    - Input: Prompt from Assess if message needs a reply1  
    - Output: GPT response JSON

  - **JSON Parser1**  
    - Type: Structured Output Parser  
    - Role: Parses GPT response JSON strictly to extract `needsReply` boolean.  
    - Configuration: JSON schema requiring `needsReply` boolean property.  
    - Input: GPT response from OpenAI Chat1  
    - Output: Parsed JSON for conditional node use.

  - **If Needs Reply1**  
    - Type: If node  
    - Role: Checks if `needsReply` is true and routes workflow accordingly.  
    - Condition: `{{$json["needsReply"]}} === true`

  - **Sticky Notes**  
    - Visual annotations describing the decision process for reply necessity.

#### 2.3 AI Response Generation with RAG

- **Overview:**  
  For emails requiring a reply, this block uses a Retrieval-Augmented Generation agent combining Pinecone vector search of a custom knowledge base with GPT-4.1 to generate an empathetic, concise, and professional email reply.

- **Nodes Involved:**  
  - AI Agent  
  - Answer questions with a vector store  
  - Pinecone Vector Store1  
  - Embeddings OpenAI  
  - OpenAI Chat Model2  
  - OpenAI Chat Model3  
  - Sticky Note3 (Custom Knowledge Base)  
  - Sticky Note4 (RAG Agent)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Main orchestrator that takes email content, uses RAG tool to search vector store, then generates the final reply.  
    - Configuration:  
      - Input Text: Subject and plain text of email from Gmail Trigger1  
      - System Message: Defines the agent as a men’s clothing store support agent who must be empathetic, professional, concise, first using RAG, and structured email format (starts with “Dear”, ends with “Best regrads”).  
      - Prompt Type: Define (custom system instructions).  
    - Inputs: Output from If Needs Reply1 (only if reply is needed)  
    - Outputs: Text reply to be sent via Gmail  
    - Failure Modes: Vector store downtime, API limits, prompt generation errors.

  - **Answer questions with a vector store**  
    - Type: LangChain Vector Store Tool  
    - Role: Uses Pinecone vector search to find relevant documents to answer user queries.  
    - Configuration: Tool named `safai_info` designated for all user questions.  
    - Inputs: Receives queries from AI Agent’s internal workflow.  
    - Outputs: Retrieved context to AI Agent for response generation.

  - **Pinecone Vector Store1**  
    - Type: Pinecone Vector Store Node  
    - Role: Connects to the Pinecone index `mens-collection` namespace for vector storage and retrieval.  
    - Credentials: Uses Pinecone API credentials.  
    - Inputs: Embeddings from OpenAI Embeddings node.  
    - Outputs: Provides vector search results to Vector Store Tool node.

  - **Embeddings OpenAI**  
    - Type: OpenAI Embeddings Node  
    - Role: Converts text into vector embeddings for Pinecone indexing and search.  
    - Credentials: OpenAI API credentials.  
    - Inputs: Text data from AI Agent or internal prompts.  
    - Outputs: Embeddings to Pinecone Vector Store.

  - **OpenAI Chat Model2 and Model3**  
    - Type: OpenAI Chat Models (GPT-4.1 nano variant)  
    - Role: Used internally by the AI Agent and Vector Store Tool for language generation and query interpretation.  
    - Credentials: OpenAI API credentials.

  - **Sticky Notes**  
    - Describe the custom knowledge base and RAG agent functionality visually.

#### 2.4 Sending the Reply

- **Overview:**  
  Sends the generated reply email back to the original sender using Gmail’s reply functionality, ensuring threading continuity.

- **Nodes Involved:**  
  - Gmail  
  - Sticky Note5 (Sending a Reply)

- **Node Details:**

  - **Gmail**  
    - Type: Gmail Node (Send Email - Reply)  
    - Role: Sends a reply email using the generated text content, linked to the original message ID to maintain email thread.  
    - Configuration:  
      - Operation: Reply  
      - Email Type: Text  
      - Message Content: From AI Agent output `output` property  
      - Message ID: From Gmail Trigger1 original email `id` field  
    - Credentials: Gmail OAuth2 credentials  
    - Potential Failures: OAuth token expiration, Gmail API quota, network errors.

  - **Sticky Note5**  
    - Marks the email sending phase.

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                     | Input Node(s)           | Output Node(s)              | Sticky Note                               |
|-------------------------------|--------------------------------------------|-----------------------------------|------------------------|-----------------------------|-------------------------------------------|
| Gmail Trigger1                | Gmail Trigger                             | Incoming email listener            | None                   | Assess if message needs a reply1 | ## Collecting Recieved Email              |
| Assess if message needs a reply1 | LangChain Chain LLM                       | Determine if reply is needed       | Gmail Trigger1, OpenAI Chat1, JSON Parser1 | If Needs Reply1              | ## Assesing whether the mail needs a reply or not |
| OpenAI Chat1                 | OpenAI Chat Model                         | GPT-4.1 model for reply assessment | Assess if message needs a reply1 | JSON Parser1                |                                           |
| JSON Parser1                 | Structured Output Parser                   | Parse GPT JSON response            | OpenAI Chat1            | Assess if message needs a reply1 |                                           |
| If Needs Reply1              | If Node                                   | Conditional routing based on reply need | Assess if message needs a reply1 | AI Agent                   | ## Selecting only the mails that need a reply |
| AI Agent                    | LangChain Agent                           | Generate reply using RAG           | If Needs Reply1          | Gmail                      | ## Custom Knowledge Base for the AI Agent<br>## RAG Agent |
| Answer questions with a vector store | LangChain Vector Store Tool              | Vector search over knowledge base | OpenAI Chat Model3, Pinecone Vector Store1 | AI Agent                   |                                           |
| Pinecone Vector Store1       | Pinecone Vector Store                      | Query Pinecone index               | Embeddings OpenAI       | Answer questions with a vector store |                                           |
| Embeddings OpenAI            | OpenAI Embeddings                          | Generate embeddings for text       | AI Agent / internal     | Pinecone Vector Store1       |                                           |
| OpenAI Chat Model2           | OpenAI Chat Model                         | Language generation for AI Agent   | AI Agent / internal     | AI Agent                   |                                           |
| OpenAI Chat Model3           | OpenAI Chat Model                         | Language model for vector tool     | AI Agent / internal     | Answer questions with a vector store |                                           |
| Gmail                       | Gmail Node                                | Send reply email                  | AI Agent                 | None                      | ## Sending a Reply                        |
| Sticky Note                 | Sticky Note                               | Visual annotation                  | None                   | None                      | ## Collecting Recieved Email              |
| Sticky Note1                | Sticky Note                               | Visual annotation                  | None                   | None                      | ## Assesing whether the mail needs a reply or not |
| Sticky Note2                | Sticky Note                               | Visual annotation                  | None                   | None                      | ## Selecting only the mails that need a reply |
| Sticky Note3                | Sticky Note                               | Visual annotation                  | None                   | None                      | ## Custom Knowledge Base for the AI Agent |
| Sticky Note4                | Sticky Note                               | Visual annotation                  | None                   | None                      | ## RAG Agent                             |
| Sticky Note5                | Sticky Note                               | Visual annotation                  | None                   | None                      | ## Sending a Reply                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**
   - Type: Gmail Trigger  
   - Credentials: Configure Gmail OAuth2 credentials for the support account.  
   - Parameters: Set filter query to exclude own emails (`q: -from:me`).  
   - Poll every minute.

2. **Create Chain LLM Node ("Assess if message needs a reply"):**
   - Type: LangChain Chain LLM  
   - Credentials: Link OpenAI API credentials.  
   - Prompt:  
     ```
     Subject: {{ $json.subject }}
     Message:
     {{ $json.textAsHtml }}
     ```
   - Messages: Instruct to output JSON `{ "needsReply": true/false }`, only true if email concerns men’s clothing store.

3. **Create OpenAI Chat Model Node:**
   - Type: OpenAI Chat Model  
   - Model: GPT-4.1  
   - Parameters: Temperature 0, Response format JSON object.  
   - Credentials: OpenAI API credentials.

4. **Create JSON Parser Node:**
   - Type: Structured Output Parser  
   - JSON Schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "needsReply": { "type": "boolean" }
       },
       "required": ["needsReply"]
     }
     ```

5. **Create If Node ("If Needs Reply"):**
   - Condition: Check if `{{$json.needsReply}} === true`  
   - Connect input from Chain LLM node.  
   - True branch leads to AI Agent node.

6. **Set up the AI Agent Node:**
   - Type: LangChain Agent  
   - Inputs: Connect from If Node true branch.  
   - Text Input:  
     ```
     Subject: {{ $('Gmail Trigger1').item.json.headers.subject }}
     Message: {{ $('Gmail Trigger1').item.json.text }}
     ```
   - System Message:  
     ```
     You are a friendly and professional support agent for a men's clothing store whose goal is to help users by providing clear, accurate, and empathetic responses to their email. When a user asks a question or describes an issue through email regarding anything related to men's clothing store, use the Retrieval-Augmented Generation (RAG) tool to search and fetch the most relevant, up-to-date information from the knowledge base or documents available. Start the response with "Dear" header and end by saying "Best regrads" footer just like how a mail is written. (Dont add any name after "Dear")

     Your responses should:

     * Acknowledge the user’s concern with empathy.
     * Use the RAG tool to find precise information relevant to the user’s query.
     * Clearly explain the solution or information in simple, easy-to-understand language.
     * If you don’t find an exact answer, be honest about it and offer helpful alternatives or next steps.
     * Always maintain a polite, patient, and supportive tone.
     * Keep responses short and concise: Only provide the price and key features of the service requested. Avoid additional details unless directly requested.
     ```
   - Prompt Type: Define

7. **Create Vector Store Tool Node ("Answer questions with a vector store"):**
   - Type: LangChain Vector Store Tool  
   - Tool Name: `safai_info`  
   - Description: Use this tool to answer every question by the user.  
   - Inputs: Connected from OpenAI Chat Model3 and Pinecone Vector Store1 outputs (internally managed by AI Agent).

8. **Create Pinecone Vector Store Node:**
   - Type: Pinecone Vector Store  
   - Credentials: Configure with Pinecone API credentials.  
   - Parameters:  
     - Index Name: `mens-collection`  
     - Namespace: `mens-collection`

9. **Create Embeddings OpenAI Node:**
   - Type: OpenAI Embeddings  
   - Credentials: OpenAI API credentials.  
   - Connect output to Pinecone Vector Store1 for embedding input.

10. **Create OpenAI Chat Model2 and Chat Model3 Nodes:**
    - Type: OpenAI Chat Model  
    - Model: GPT-4.1 Nano  
    - Credentials: OpenAI API credentials.  
    - Used internally by AI Agent and Vector Store Tool.

11. **Create Gmail Send Node:**
    - Type: Gmail  
    - Credentials: Same Gmail OAuth2 credentials.  
    - Operation: Reply  
    - Email Type: Text  
    - Message: Set to `={{ $json.output }}` from AI Agent node.  
    - Message ID: Use `={{ $('Gmail Trigger1').item.json.id }}` to reply in thread.

12. **Connect Nodes:**
    - Gmail Trigger1 → Assess if message needs a reply1  
    - Assess if message needs a reply1 → OpenAI Chat1 → JSON Parser1 → If Needs Reply1  
    - If Needs Reply1 (True) → AI Agent → Gmail  
    - AI Agent uses internal connections to Vector Store Tool, Pinecone Vector Store, Embeddings OpenAI, OpenAI Chat Models as configured.

13. **Add Sticky Notes:**
    - Add notes to mark phases: Collecting Received Email, Assessing Reply Need, Selecting Emails for Reply, Custom Knowledge Base, RAG Agent, and Sending Reply for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4.1 with a nano variant for faster, cost-efficient responses in vector search and generation.                                                        | OpenAI GPT-4.1 nano model usage in LangChain nodes.                                               |
| The RAG approach integrates Pinecone as vector knowledge base, enabling up-to-date, precise responses from store-specific documents.                                        | Pinecone vector store setup and retrieval.                                                        |
| Gmail OAuth2 credentials must have proper scopes to read, filter, and send emails on behalf of the support account.                                                         | Gmail API OAuth2 setup in n8n.                                                                    |
| The system message in the AI Agent carefully instructs tone and formatting to ensure professional, empathetic customer support emails.                                    | System prompt best practice for customer support AI.                                              |
| Sticky notes visually segment the workflow for better maintainability and understanding.                                                                                   | Workflow annotations in n8n editor.                                                               |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.