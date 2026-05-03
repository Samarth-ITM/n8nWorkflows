Customer Support & Lead Collection Chatbot with RAG, GPT-4o, Sheets & Telegram

https://n8nworkflows.xyz/workflows/customer-support---lead-collection-chatbot-with-rag--gpt-4o--sheets---telegram-9234


# Customer Support & Lead Collection Chatbot with RAG, GPT-4o, Sheets & Telegram

### 1. Workflow Overview

This workflow implements a **Customer Support & Lead Collection Chatbot** using Retrieval-Augmented Generation (RAG) with GPT-4o, Pinecone vector search, Google Sheets, and Telegram integration. It is designed for businesses to provide automated, knowledgeable customer interaction while capturing potential leads for sales follow-up.

**Target Use Cases:**

- SMBs, startups, agencies, or internal teams wanting a 24/7 chatbot that:  
  - Answers company-related questions with up-to-date knowledge  
  - Collects customer contact details (name, email, phone, interests) after interaction  
  - Stores leads centrally in Google Sheets  
  - Sends instant Telegram notifications to the team  
- Use cases requiring smooth conversational memory and multi-tool orchestration.

**Logical Blocks:**

- **1.1 Input Reception:** Receives chat messages from users via the LangChain Chat Trigger node.  
- **1.2 AI Processing & Memory:** The AI Agent node manages conversation flow using GPT-4o, linked with conversation memory and multiple tools.  
- **1.3 Knowledge Retrieval (RAG):** Retrieves company-specific answers from a Pinecone vector store populated with company knowledge via embeddings.  
- **1.4 Lead Capture & Notification:** After answering queries, the bot collects lead info, appends it to Google Sheets, and sends a notification via Telegram.  
- **1.5 Supporting Models & Tools:** Dedicated nodes for embeddings, vector store connection, and the company answering model.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives incoming chat messages from users and triggers the workflow.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **When chat message received**  
  - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
  - Role: Entry point webhook triggered when a user sends a message.  
  - Configuration: Default options; webhook ID assigned for external chat interface integration.  
  - Inputs: External chat interface/webhook.  
  - Outputs: Passes the chat message to the AI Agent node.  
  - Edge cases: Webhook downtime or misconfiguration may cause missed messages.

---

#### 2.2 AI Processing & Memory

**Overview:**  
Handles the AI conversational logic using GPT-4o, maintains conversation context, and orchestrates tool usage.

**Nodes Involved:**  
- AI Agent  
- Conversation Memory  
- Main Chat Model

**Node Details:**  
- **AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Core orchestrator managing conversation, tool selection, and business logic.  
  - Configuration:  
    - System message sets the assistant’s role as a friendly company helper.  
    - Defines three tools: Company Q&A, Google Sheets Agentic Tool, Telegram Agentic Tool.  
    - After answering common questions, prompts user for contact info.  
  - Inputs: Receives chat messages from the "When chat message received" node.  
  - Outputs: Delegates to tools and returns responses.  
  - Edge cases: Expression or prompt failures if placeholders (e.g. company name) are not set; API rate limits; unexpected user inputs.

- **Conversation Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains the last 12 interaction turns to provide context continuity.  
  - Configuration: Context window length set to 12 messages.  
  - Inputs: Connected as memory source for the AI Agent.  
  - Outputs: Provides context to the Agent for better responses.  
  - Edge cases: Excessively long conversations might exceed context window, losing earlier context.

- **Main Chat Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: Language model (GPT-4o) for general conversation generation.  
  - Configuration: Model set to GPT-4o, no additional special options.  
  - Inputs: Used by AI Agent as the primary language model.  
  - Outputs: Generates chatbot replies.  
  - Edge cases: API key issues, timeouts, or model unavailability.

---

#### 2.3 Knowledge Retrieval (RAG)

**Overview:**  
Uses a Pinecone vector store to retrieve relevant company knowledge embeddings to provide accurate answers.

**Nodes Involved:**  
- Company Q&A  
- Pinecone Vector Store (Company KB)  
- Generate Embeddings (OpenAI)  
- Company Answering Model

**Node Details:**  
- **Company Q&A**  
  - Type: `@n8n/n8n-nodes-langchain.toolVectorStore`  
  - Role: Acts as a tool for the AI Agent to query company knowledge.  
  - Configuration: Described as answering company-related questions.  
  - Inputs: Receives vector store results and language model outputs.  
  - Outputs: Returns relevant answers to the AI Agent.  
  - Edge cases: Empty or outdated knowledge base can give poor answers.

- **Pinecone Vector Store (Company KB)**  
  - Type: `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  - Role: Connects to Pinecone index 'company' in namespace 'Q&A'.  
  - Configuration: Uses Pinecone API credentials and namespace 'Q&A'.  
  - Inputs: Receives embeddings from the Generate Embeddings node.  
  - Outputs: Returns relevant document vectors to the Company Q&A node.  
  - Edge cases: Pinecone API errors, namespace/index misconfiguration.

- **Generate Embeddings (OpenAI)**  
  - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  - Role: Generates text embeddings using OpenAI's `text-embedding-3-small` or default embedding model.  
  - Configuration: Credentials for OpenAI API provided.  
  - Inputs: Receives text queries for embedding generation.  
  - Outputs: Embeddings sent to Pinecone Vector Store.  
  - Edge cases: API limits, model deprecation.

- **Company Answering Model**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: GPT-4o used specifically for answering based on retrieved knowledge.  
  - Configuration: Model set to GPT-4o.  
  - Inputs: Receives context from retrieved documents.  
  - Outputs: Generates concise, relevant answers for the Company Q&A tool.  
  - Edge cases: Same as other GPT-4o nodes (timeouts, API errors).

---

#### 2.4 Lead Capture & Notification

**Overview:**  
After delivering support answers, the workflow collects lead information and stores it centrally, then notifies the team on Telegram.

**Nodes Involved:**  
- Save Lead to Google Sheets  
- Send Lead to Telegram

**Node Details:**  
- **Save Lead to Google Sheets**  
  - Type: `n8n-nodes-base.googleSheetsTool`  
  - Role: Appends lead contact information to a Google Sheet.  
  - Configuration:  
    - Maps columns: Name, Email, Phone, Interested in  
    - Uses specific Google Sheet ID and Sheet tab (`gid=0`).  
    - OAuth2 credentials configured for access.  
  - Inputs: Receives lead data from AI Agent outputs via expressions.  
  - Outputs: Confirms data appended; no further output nodes connected.  
  - Edge cases: OAuth token expiration, Google Sheets API quota, malformed data.

- **Send Lead to Telegram**  
  - Type: `n8n-nodes-base.telegramTool`  
  - Role: Sends a Telegram message notification with lead summary and contact info.  
  - Configuration:  
    - Chat ID must be set (replace `[INSERT_YOUR_CHAT_ID]`).  
    - Message text dynamically set from AI Agent output.  
    - Telegram API credentials configured.  
  - Inputs: Lead summary text from AI Agent.  
  - Outputs: Confirmation of message sent.  
  - Edge cases: Invalid chat ID, Telegram API limits, bot permission issues.

---

#### 2.5 Supporting Nodes: Sticky Notes & Documentation

**Overview:**  
Provides user guidance and documentation within the workflow canvas.

**Nodes Involved:**  
- Trigger – How to Publish  
- AI Agent – Flow  
- RAG Database – Setup  
- Template Guide

**Node Details:**  
- All sticky notes contain explanatory text about different workflow aspects, including deployment instructions, AI agent overview, RAG setup, and full template guide.  
- They do not affect workflow execution but provide essential context for users customizing or deploying the workflow.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                            | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                             |
|-----------------------------|-----------------------------------|--------------------------------------------|--------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received  | @n8n/n8n-nodes-langchain.chatTrigger | Entry point webhook for incoming messages  | (External chat interface)       | AI Agent                       | Trigger – How to Publish: "You can switch to Publicly Available to share this chatbot interface."      |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent    | Core AI orchestrator managing tools & dialogue | When chat message received      | Company Q&A, Save Lead to Google Sheets, Send Lead to Telegram | AI Agent – Flow: "Powered with GPT-4o; collects leads; sends Telegram notifications."                   |
| Conversation Memory         | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains short-term conversation context  | (Connected as AI Agent memory)  | AI Agent                       | AI Agent – Flow (same as AI Agent node)                                                                |
| Main Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model for general conversation    | (Connected as AI Agent model)   | AI Agent                       | AI Agent – Flow                                                                                         |
| Company Q&A                 | @n8n/n8n-nodes-langchain.toolVectorStore | Tool for answering company questions via RAG | Pinecone Vector Store (Company KB), Company Answering Model | AI Agent                       | RAG Database – Setup: "Connects Pinecone RAG database with company knowledge."                         |
| Pinecone Vector Store (Company KB) | @n8n/n8n-nodes-langchain.vectorStorePinecone | Connects to Pinecone index and namespace   | Generate Embeddings (OpenAI)    | Company Q&A                   | RAG Database – Setup                                                                                     |
| Generate Embeddings (OpenAI) | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for vector store      | (Text input from queries)       | Pinecone Vector Store (Company KB) | RAG Database – Setup                                                                                     |
| Company Answering Model     | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model to answer based on retrieved docs | Company Q&A                    | Company Q&A                   | RAG Database – Setup                                                                                     |
| Save Lead to Google Sheets  | n8n-nodes-base.googleSheetsTool   | Stores collected lead info in Google Sheets | AI Agent                      | (End)                         | AI Agent – Flow                                                                                         |
| Send Lead to Telegram       | n8n-nodes-base.telegramTool        | Sends Telegram notification with lead info | AI Agent                      | (End)                         | AI Agent – Flow                                                                                         |
| Trigger – How to Publish    | n8n-nodes-base.stickyNote          | Documentation about webhook publishing      | -                              | -                             | "You can switch to Publicly Available to share this chatbot interface."                                |
| AI Agent – Flow             | n8n-nodes-base.stickyNote          | Documentation describing AI Agent behavior  | -                              | -                             | "Powered with GPT-4o; collects leads; sends Telegram notifications."                                  |
| RAG Database – Setup        | n8n-nodes-base.stickyNote          | Documentation for RAG setup with Pinecone   | -                              | -                             | "Connect Pinecone RAG database for company knowledge."                                                |
| Template Guide              | n8n-nodes-base.stickyNote          | Full template overview, prerequisites, and customization guide | -                    | -                             | Detailed guide on workflow purpose, usage, prerequisites, and customization instructions.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add `@n8n/n8n-nodes-langchain.chatTrigger` node.  
   - Configure webhook to receive chat messages externally. Assign webhookId or leave default.  

2. **Add the AI Agent Node:**  
   - Add `@n8n/n8n-nodes-langchain.agent` node.  
   - Set system message:  
     ```
     ## Role:
     You are a friendly assistant for a company named **[INSERT_YOUR_COMPANY_NAME_HERE]**.

     ## Task:
     You answer questions about the business.

     ## Details:
     You have access to various tools, which you use correctly.

     ## Tools:
     - Company Q&A  
     - Google Sheets Agentic Tool  
     - Telegram Agentic Tool

     After a customer asks about opening hours, products, location, or business information, ask them for their name, email, specific interests and phone number.
     ```  
   - Connect input from the chat trigger node.  

3. **Add Conversation Memory Node:**  
   - Add `@n8n/n8n-nodes-langchain.memoryBufferWindow` node.  
   - Set context window length to 12.  
   - Connect this node as the `ai_memory` input to the AI Agent node.  

4. **Add Main Chat Model Node:**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node.  
   - Set model to `gpt-4o`.  
   - Connect as `ai_languageModel` input to AI Agent node.  
   - Configure OpenAI API credentials.  

5. **Set up Company Q&A Tool:**  
   - Add `@n8n/n8n-nodes-langchain.toolVectorStore` node.  
   - Set description: "Gives answers related to the company."  
   - Connect as `ai_tool` input to AI Agent node.  

6. **Add Pinecone Vector Store Node:**  
   - Add `@n8n/n8n-nodes-langchain.vectorStorePinecone` node.  
   - Configure with Pinecone API credentials.  
   - Set index name to your Pinecone index (e.g., "company").  
   - Set namespace to "Q&A".  
   - Connect `ai_vectorStore` output to Company Q&A node.  

7. **Add Generate Embeddings Node:**  
   - Add `@n8n/n8n-nodes-langchain.embeddingsOpenAi` node.  
   - Configure OpenAI API credentials.  
   - Connect `ai_embedding` output to Pinecone Vector Store node.  

8. **Add Company Answering Model:**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node.  
   - Set model to `gpt-4o`.  
   - Configure OpenAI API credentials.  
   - Connect as `ai_languageModel` input to Company Q&A node.  

9. **Add Save Lead to Google Sheets Node:**  
   - Add `n8n-nodes-base.googleSheetsTool` node.  
   - Set operation to `append`.  
   - Configure spreadsheet ID and sheet name/tab (e.g., gid=0).  
   - Map columns: Name, Email, Phone, Interested in using expressions from AI Agent output.  
   - Connect as `ai_tool` input to AI Agent node.  
   - Configure Google Sheets OAuth2 credentials.  

10. **Add Send Lead to Telegram Node:**  
    - Add `n8n-nodes-base.telegramTool` node.  
    - Set `chatId` to your Telegram chat/group ID.  
    - Set message text dynamically from AI Agent output (lead summary).  
    - Connect as `ai_tool` input to AI Agent node.  
    - Configure Telegram API credentials.  

11. **Connect Workflow:**  
    - Connect "When chat message received" → "AI Agent".  
    - Connect "Conversation Memory" → AI Agent memory input.  
    - Connect "Main Chat Model" → AI Agent language model input.  
    - Connect "Company Q&A" → AI Agent tool input.  
    - Connect "Save Lead to Google Sheets" → AI Agent tool input.  
    - Connect "Send Lead to Telegram" → AI Agent tool input.  
    - Connect "Generate Embeddings" → "Pinecone Vector Store" → "Company Q&A" → "Company Answering Model".  

12. **Final Checks:**  
    - Replace placeholders:  
      - `[INSERT_YOUR_COMPANY_NAME_HERE]` in AI Agent system message  
      - Pinecone index and namespace with your own data  
      - Google Sheets document ID and sheet/tab name  
      - Telegram chat ID  
    - Ensure all API credentials are valid and authorized.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| You can switch the chat trigger webhook to "Publicly Available" mode to share the chatbot interface via a public link.                                                                                                                                            | Trigger – How to Publish sticky note                                                                                |
| The AI Agent uses GPT-4o by default but can be replaced with any LLM supported by LangChain.                                                                                                                                                                         | AI Agent – Flow sticky note                                                                                        |
| The RAG database uses Pinecone with OpenAI embeddings (`text-embedding-3-small`) for semantic search of company knowledge.                                                                                                                                        | RAG Database – Setup sticky note                                                                                   |
| Leads are stored in Google Sheets with columns: Name, Email, Phone, Interested in. Telegram notifications include a summary and contact details.                                                                                                                  | AI Agent – Flow sticky note                                                                                        |
| Keep your knowledge base concise and up-to-date for better RAG accuracy; consider adding duplicate checks or CRM integration as enhancements.                                                                                                                     | Template Guide sticky note                                                                                         |
| This workflow was created by n8n Creators and is designed for SMBs and internal teams to automate repetitive Q&A and lead capture.                                                                                                                                | Tag: n8n Creators                                                                                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. All content complies strictly with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.