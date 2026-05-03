Automated Gmail Support Agent with Gemini 2.5, RAG & Cohere Reranking

https://n8nworkflows.xyz/workflows/automated-gmail-support-agent-with-gemini-2-5--rag---cohere-reranking-9382


# Automated Gmail Support Agent with Gemini 2.5, RAG & Cohere Reranking

### 1. Workflow Overview

This workflow automates email support for an agency using Gmail as the input source and Google Gemini 2.5 as the AI language model. It integrates Retrieval-Augmented Generation (RAG) with a Pinecone vector store containing agency knowledge, enhanced by semantic reranking with Cohere for improved relevance. A LangChain-based AI agent processes incoming emails, accesses contextual memory stored in Postgres, retrieves and reranks relevant documents, and generates customized email replies automatically sent back via Gmail.

**Logical blocks:**

- **1.1 Input Reception**: Gmail Trigger node listens for new incoming emails.
- **1.2 AI Processing & Memory**: The Email Support Agent node uses Gemini 2.5 LLM and Postgres memory to handle email content contextually.
- **1.3 Knowledge Retrieval and Reranking**: Pinecone Retriever fetches relevant documents from the agency knowledge base using OpenAI embeddings, and Cohere Reranker improves semantic relevance.
- **1.4 Response Generation and Output**: Gemini 2.5 generates reply text, which the Gmail Reply node sends back to the original sender.
- **1.5 Notes and Documentation**: Sticky Notes provide explanations and configuration guidance.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block waits for new emails arriving in the configured Gmail account, triggering the workflow to start processing.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**

- **Gmail Trigger**  
  - *Type & Role:* Event trigger node for new Gmail emails.  
  - *Configuration:* Polls every minute for new emails without any filters (all emails trigger).  
  - *Expressions/Variables:* N/A  
  - *Connections:* Output connected to Email Support Agent node input.  
  - *Version Requirements:* n8n Gmail Trigger node v1.3+ recommended.  
  - *Edge Cases / Failures:*  
    - Gmail API quota or auth errors.  
    - Delays or missed triggers due to polling interval.  
  - *Credentials:* Gmail OAuth2 with read and send scopes.

---

#### 2.2 AI Processing & Memory

**Overview:**  
Processes the incoming email content through an AI agent using Gemini 2.5 LLM, enriched with Pinecone knowledge retrieval and Postgres memory to maintain conversational context per email.

**Nodes Involved:**  
- Email Support Agent  
- Gemini 2.5  
- Postgres Memory

**Node Details:**

- **Email Support Agent**  
  - *Type & Role:* LangChain AI agent node orchestrating AI model, tools, and memory.  
  - *Configuration:*  
    - Accepts email text as input.  
    - Uses Gemini 2.5 as the language model.  
    - Uses Pinecone Retriever as a tool.  
    - Uses Postgres chat memory keyed by email ID.  
    - System message defines agent as an email support agent for the agency.  
  - *Expressions/Variables:* Input text from Gmail Trigger email body (`{{$json.text}}`) and email ID for memory session key (`{{$json.id}}`).  
  - *Connections:* Receives input from Gmail Trigger; outputs reply text to Gmail Reply node.  
  - *Version Requirements:* LangChain Agent node v2.2+.  
  - *Edge Cases / Failures:*  
    - LLM API authentication or rate-limit errors.  
    - Memory session key missing or Postgres connection failure.  
    - Expression evaluation failure if email fields missing.  
  - *Credentials:* None directly, but relies on downstream node credentials.

- **Gemini 2.5**  
  - *Type & Role:* Language model node using Google Gemini 2.5 LLM.  
  - *Configuration:* Default options, connected as AI language model for Email Support Agent.  
  - *Expressions/Variables:* None directly; receives prompts from the agent.  
  - *Connections:* Linked as AI model input to Email Support Agent.  
  - *Version Requirements:* Gemini 2.5 node v1+.  
  - *Edge Cases / Failures:* API authentication or throttling errors.  
  - *Credentials:* Google PaLM API key (Google Gemini).

- **Postgres Memory**  
  - *Type & Role:* Memory node for LangChain agent; stores conversation state in Postgres.  
  - *Configuration:*  
    - Table name: "email_support_agent_"  
    - Session key: uses email ID as custom key for session.  
  - *Expressions/Variables:* Session key expression (`{{$json.id}}`) from email.  
  - *Connections:* Linked as memory input to Email Support Agent.  
  - *Version Requirements:* Postgres memory node v1.3+.  
  - *Edge Cases / Failures:*  
    - Postgres connection/auth failures.  
    - Missing or invalid session keys.  
  - *Credentials:* Postgres database connection.

---

#### 2.3 Knowledge Retrieval and Reranking

**Overview:**  
Retrieves relevant documents from the agency’s knowledge base stored in Pinecone using OpenAI embeddings, then reranks these results with Cohere to improve semantic relevance before feeding back into the AI agent.

**Nodes Involved:**  
- Pinecone Retriever  
- OpenAI Embeddings  
- Cohere Reranker

**Node Details:**

- **Pinecone Retriever**  
  - *Type & Role:* Vector store retriever node querying Pinecone index for knowledge documents.  
  - *Configuration:*  
    - Index: "agency-info" (pre-populated with agency documents).  
    - Retrieve mode: "retrieve-as-tool" for LangChain integration.  
    - Top K: 10 results.  
    - Use Reranker enabled, connected to Cohere node.  
    - Tool description includes context about agency services and FAQs.  
  - *Expressions/Variables:* None specific, uses upstream embeddings.  
  - *Connections:* Receives embeddings from OpenAI Embeddings node; outputs to Email Support Agent as a tool; reranked by Cohere.  
  - *Version Requirements:* Pinecone node v1.3+.  
  - *Edge Cases / Failures:*  
    - Pinecone API auth or network errors.  
    - Empty or improperly indexed data returns no results.  
    - Mismatch in embedding dimension or index schema.  
  - *Credentials:* Pinecone API key.

- **OpenAI Embeddings**  
  - *Type & Role:* Generates vector embeddings for text queries using OpenAI.  
  - *Configuration:*  
    - Model: "text-embedding-3-large" with 1024 dimensions.  
  - *Expressions/Variables:* Passes embeddings to Pinecone Retriever.  
  - *Connections:* Output connected to Pinecone Retriever input.  
  - *Version Requirements:* OpenAI Embeddings node v1.2+.  
  - *Edge Cases / Failures:*  
    - API key invalid or rate limits.  
    - Embedding computation errors.  
  - *Credentials:* OpenAI API key.

- **Cohere Reranker**  
  - *Type & Role:* Semantic reranker node that reorders Pinecone results for improved relevance.  
  - *Configuration:* Default, connected to Pinecone Retriever as reranker.  
  - *Expressions/Variables:* None directly.  
  - *Connections:* Input from Pinecone Retriever; output back to Pinecone Retriever for reranking.  
  - *Version Requirements:* Cohere Reranker node v1+.  
  - *Edge Cases / Failures:*  
    - Cohere API key invalid or quota exceeded.  
    - Token usage monitoring required.  
  - *Credentials:* Cohere API key.

---

#### 2.4 Response Generation and Output

**Overview:**  
The AI-generated reply from Gemini 2.5 is sent back as an email reply to the original sender using the Gmail Reply node.

**Nodes Involved:**  
- Gmail Reply

**Node Details:**

- **Gmail Reply**  
  - *Type & Role:* Sends an email reply via Gmail API.  
  - *Configuration:*  
    - Reply message: uses the output text from Email Support Agent `{{$json.output}}`.  
    - Message ID: replies to the original email using the Gmail Trigger’s email ID `{{$('Gmail Trigger').item.json.id}}`.  
    - Options: disables appended attribution (no signature appended by Gmail).  
    - Email type: plain text.  
  - *Expressions/Variables:* References Email Support Agent output and Gmail Trigger message ID.  
  - *Connections:* Input from Email Support Agent node output.  
  - *Version Requirements:* Gmail node v2.1+ recommended.  
  - *Edge Cases / Failures:*  
    - Gmail API authentication or quota errors.  
    - Reply fails if original message ID missing or invalid.  
  - *Credentials:* Gmail OAuth2 with send scopes.

---

#### 2.5 Notes and Documentation

**Overview:**  
Sticky Notes provide detailed explanations, configuration steps, and troubleshooting tips for the entire workflow and key nodes.

**Nodes Involved:**  
- Overview Note1  
- Note: Agent  
- Note: Pinecone  
- Note: Cohere

**Node Details:**

- **Overview Note1**  
  - *Content:* Full workflow description, prerequisites, credential setup instructions, configuration steps, use cases, and troubleshooting tips.  
  - *Position:* Top-left for global context.  
  - *Edge Cases:* N/A

- **Note: Agent**  
  - *Content:* Explains Email Support Agent’s purpose and configuration using Gemini 2.5 and Postgres memory.  
  - *Position:* Near Email Support Agent node.

- **Note: Pinecone**  
  - *Content:* Details on Pinecone Retriever’s role, settings, and importance of prepopulated knowledge base.  
  - *Position:* Near Pinecone Retriever node.

- **Note: Cohere**  
  - *Content:* Describes Cohere Reranker’s semantic reranking function and API key management.  
  - *Position:* Near Cohere Reranker node.

---

### 3. Summary Table

| Node Name          | Node Type                              | Functional Role                               | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                          |
|--------------------|--------------------------------------|-----------------------------------------------|----------------------|----------------------|----------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger      | Gmail Trigger (Event)                 | Triggers workflow on new Gmail email           | -                    | Email Support Agent   |                                                                                                                      |
| Email Support Agent| LangChain Agent                      | AI agent processing email with tools & memory | Gmail Trigger        | Gmail Reply           | ## 🤖 Node: Email Support Agent<br>AI agent processes email with tools/memory. LLM: Gemini 2.5, Memory: Postgres.      |
| Gemini 2.5         | LangChain LLM Node                   | Language model generating AI text              | Email Support Agent (ai_languageModel) | Email Support Agent |                                                                                                                      |
| Postgres Memory    | LangChain Memory Node                | Stores conversational memory per email         | -                    | Email Support Agent   |                                                                                                                      |
| Pinecone Retriever | Vector Store Retriever (Pinecone)   | Retrieves top 10 docs from agency knowledge base | OpenAI Embeddings, Cohere Reranker | Email Support Agent (ai_tool) | ## 🗄️ Node: Pinecone Retriever<br>RAG tool retrieves top 10 docs with OpenAI embeddings & Cohere reranker enabled.      |
| OpenAI Embeddings  | Embeddings Node (OpenAI)             | Generates text embeddings for retrieval         | -                    | Pinecone Retriever    |                                                                                                                      |
| Cohere Reranker    | Reranker Node (Cohere)               | Semantically reranks Pinecone results           | Pinecone Retriever   | Pinecone Retriever    | ## 🔄 Node: Cohere Reranker<br>Semantically reorders Pinecone results for better relevance. Monitor token usage.       |
| Gmail Reply        | Gmail Node (Send email reply)        | Sends AI-generated reply to original email     | Email Support Agent   | -                    |                                                                                                                      |
| Overview Note1     | Sticky Note                         | Workflow overview, prerequisites, troubleshooting | -                    | -                    | See detailed workflow description, credential setup, use cases, and troubleshooting in the note content.               |
| Note: Agent        | Sticky Note                         | Explains Email Support Agent node               | -                    | -                    | See explanation on Email Support Agent functionality and configuration.                                              |
| Note: Pinecone     | Sticky Note                         | Explains Pinecone Retriever node                 | -                    | -                    | See explanation on Pinecone Retriever details and pre-population requirement.                                        |
| Note: Cohere       | Sticky Note                         | Explains Cohere Reranker node                     | -                    | -                    | See explanation on Cohere Reranker role and API usage monitoring.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Set polling to "Every Minute" without filters.  
   - Assign Gmail OAuth2 credentials with scopes: `gmail.readonly` and `gmail.send`.  

2. **Create Email Support Agent node**  
   - Type: LangChain Agent  
   - Set input text parameter to `{{$json.text}}` (email body from Gmail Trigger).  
   - Set system message to: "You are an email support agent for the agency. Use the pinecone tool to access its knowledge base."  
   - Configure prompt type as "define".  
   - Connect this node’s main input from Gmail Trigger output.  

3. **Add Gemini 2.5 node**  
   - Type: LangChain LLM (Google Gemini 2.5)  
   - Configure with Google PaLM API credentials.  
   - Use default options.  
   - Connect Gemini 2.5 as the AI language model input to Email Support Agent (ai_languageModel input).  

4. **Add Postgres Memory node**  
   - Type: LangChain Postgres Chat Memory  
   - Configure tableName as `"email_support_agent_"`.  
   - Use the email ID from Gmail Trigger as session key: `{{$json.id}}`.  
   - Connect Postgres Memory node as AI memory input to Email Support Agent.  
   - Assign Postgres credentials with proper connection details.  

5. **Configure Knowledge Retrieval:**  
   - Create OpenAI Embeddings node:  
     - Model: `text-embedding-3-large`  
     - Dimensions: 1024  
     - Assign OpenAI API credentials.  
   - Create Pinecone Retriever node:  
     - Mode: "retrieve-as-tool"  
     - Top K: 10  
     - Pinecone Index: select or create "agency-info" index with dimension 1024.  
     - Tool description: "Info About <agency> services, owner and general FAQ".  
     - Enable "Use Reranker" option.  
     - Connect OpenAI Embeddings output to Pinecone Retriever input.  
     - Assign Pinecone API credentials.  
   - Create Cohere Reranker node:  
     - Default settings.  
     - Assign Cohere API credentials.  
     - Connect Cohere Reranker input from Pinecone Retriever’s reranker input, output back to Pinecone Retriever reranker output.  
   - Connect Pinecone Retriever as AI tool input to Email Support Agent.  

6. **Create Gmail Reply node**  
   - Type: Gmail node (send email)  
   - Set operation to "reply".  
   - Set message content to the Email Support Agent output: `{{$json.output}}`.  
   - Set messageId to reply to original email: `{{$('Gmail Trigger').item.json.id}}`.  
   - Disable appended attribution.  
   - Set email type to "text".  
   - Assign Gmail OAuth2 credentials with send scope.  
   - Connect input from Email Support Agent main output.  

7. **Add Sticky Notes (Optional but Recommended)**  
   - Add notes for overview, agent explanation, Pinecone Retriever, and Cohere Reranker as per documentation for maintainability and onboarding.  

8. **Test and Activate Workflow**  
   - Ensure all credentials are correctly configured and valid.  
   - Populate Pinecone "agency-info" index with agency documents using a separate workflow before activating.  
   - Test with sample incoming emails to verify end-to-end processing and replies.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow triggers on new Gmail emails and automates replies using AI with RAG and semantic reranking. Requires credential setup for Gmail, OpenAI, Cohere, Google Gemini, Pinecone, and Postgres.                                            | Overview Note1 in workflow.                                                                         |
| Gmail OAuth2 setup requires enabling Gmail API in Google Cloud and using scopes: `https://www.googleapis.com/auth/gmail.readonly` and `https://www.googleapis.com/auth/gmail.send`.                                                           | Overview Note1 instructions.                                                                        |
| Pinecone index "agency-info" must be prepopulated with agency-specific knowledge base documents with 1024-dimensional embeddings matching the OpenAI embedding model used.                                                                   | Note: Pinecone and Overview Note1.                                                                 |
| Cohere API token usage should be monitored to avoid unexpected quota exhaustion due to frequent reranking requests.                                                                                                                           | Note: Cohere.                                                                                       |
| Postgres memory uses Neon or Supabase-like managed DB for chat session storage keyed by email message ID.                                                                                                                                    | Overview Note1 and Postgres Memory node.                                                           |
| Google Gemini 2.5 is used as the primary LLM; API key must be generated via https://aistudio.google.com/ and provided as Google PaLM credentials in n8n.                                                                                      | Overview Note1.                                                                                    |
| Troubleshooting tips include checking Gmail API scopes, Pinecone index population, Cohere API key validity, and Postgres connection health.                                                                                                  | Overview Note1.                                                                                    |

---

This comprehensive reference enables understanding, maintenance, and full reproduction of the Automated Gmail Support Agent workflow integrating Gemini 2.5, RAG, and Cohere reranking.