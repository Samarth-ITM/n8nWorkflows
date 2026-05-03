Build an Intelligent Q&A Bot with Lookio Knowledge Base and GPT

https://n8nworkflows.xyz/workflows/build-an-intelligent-q-a-bot-with-lookio-knowledge-base-and-gpt-8787


# Build an Intelligent Q&A Bot with Lookio Knowledge Base and GPT

---

### 1. Workflow Overview

This workflow implements an intelligent Question & Answer (Q&A) chatbot that leverages a custom knowledge base built and hosted on Lookio, combined with an OpenAI GPT model for natural language understanding and response generation. The primary use case is to provide users with accurate, context-aware answers by querying a company-specific knowledge base, while handling simple conversational interactions locally to optimize API usage and performance.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users in real-time.
- **1.2 AI Processing:** Uses an AI agent to interpret the user’s query, decide if it requires knowledge base consultation, and generate responses.
- **1.3 Knowledge Base Query:** When necessary, calls the Lookio knowledge base API to retrieve specific answers.
- **1.4 Memory Management:** Maintains conversational context using a sliding window memory buffer.
- **1.5 AI Model Execution:** Uses OpenAI’s GPT-4.1-mini chat model to generate natural language replies based on context and knowledge base outputs.
- **1.6 Workflow Annotations:** Sticky notes provide configuration instructions and usage tips for maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming chat messages via a webhook and triggers the workflow whenever a message is received.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger` (Webhook Trigger Node)  
    - Role: Entry point for the workflow; triggers processing upon new chat input.  
    - Configuration: Uses an auto-generated webhook ID to listen for incoming chat messages. No additional options configured.  
    - Expressions/Variables: Incoming chat message data is passed downstream for processing.  
    - Input: None (Webhook trigger)  
    - Output: Main output connected to AI Knowledge Agent node.  
    - Edge cases: Possible webhook misconfiguration, message payload format errors, or webhook downtime.  
    - Version: 1.3  

#### 2.2 AI Processing

- **Overview:**  
  Acts as the central controller for interpreting input messages, deciding when to query the knowledge base, and generating final replies.

- **Nodes Involved:**  
  - AI Knowledge Agent

- **Node Details:**

  - **AI Knowledge Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent` (AI Agent Node)  
    - Role: Processes user queries intelligently by deciding whether to answer directly or call external tools.  
    - Configuration:  
      - System message instructs it to answer queries from the knowledge base when needed, and transparently notify the user if knowledge is insufficient.  
      - Uses a single tool named “Query knowledge base” to fetch answers.  
    - Expressions: Uses system message with multi-line string to control agent behavior.  
    - Input: Receives chat message from “When chat message received.”  
    - Output: Sends requests to OpenAI Chat Model, Simple Memory, and Query knowledge base nodes via dedicated connections (ai_languageModel, ai_memory, ai_tool).  
    - Edge cases: Potential API rate limits, misinterpretation of queries, or failure if knowledge base tool returns unexpected data.  
    - Version: 2.2  

#### 2.3 Knowledge Base Query

- **Overview:**  
  Queries the Lookio knowledge base API to retrieve answers for user questions requiring specific knowledge.

- **Nodes Involved:**  
  - Query knowledge base

- **Node Details:**

  - **Query knowledge base**  
    - Type: `n8n-nodes-base.httpRequestTool` (HTTP Request Tool Node)  
    - Role: Makes POST requests to Lookio’s API to get knowledge base answers.  
    - Configuration:  
      - URL: `https://api.lookio.app/webhook/query`  
      - Method: POST  
      - Body parameters include:  
        - `query`: dynamically set from AI agent input (user’s question) via expression using `$fromAI`.  
        - `assistant_id`: placeholder for Lookio assistant ID (must be replaced by user).  
        - `query_mode`: set to `"flash"` for response mode.  
      - Header includes an API key placeholder `<your-lookio-api-key>` for authentication.  
      - Tool description clarifies its dedicated use to answer knowledge-based questions.  
    - Input: Receives query parameter from AI Knowledge Agent node.  
    - Output: Returns knowledge base response to AI Knowledge Agent.  
    - Edge cases: HTTP errors, invalid API keys, malformed queries, network timeouts, or API rate limits.  
    - Version: 4.2  

#### 2.4 Memory Management

- **Overview:**  
  Maintains conversational context in a sliding window memory buffer to provide the AI agent with relevant recent chat history.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Memory Buffer Node)  
    - Role: Stores recent interactions in a buffer to maintain context for the AI agent.  
    - Configuration: Default parameters, implying a standard window size and behavior.  
    - Input: Connected from AI Knowledge Agent’s ai_memory output.  
    - Output: Sends memory context back to AI Knowledge Agent’s ai_memory input.  
    - Edge cases: Memory overflow, loss of context if buffer size too small, or stale data if buffer size too large.  
    - Version: 1.3  

#### 2.5 AI Model Execution

- **Overview:**  
  Interfaces with OpenAI’s GPT-4.1-mini chat model to generate natural language responses based on the AI agent’s processed input and memory.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Node)  
    - Role: Generates chat completions from the GPT-4.1-mini model.  
    - Configuration:  
      - Model selected: `gpt-4.1-mini` (a lighter GPT-4 variant).  
      - No additional options configured.  
    - Credentials: Uses OpenAI API key credential named “Duv's OpenAI.”  
    - Input: Connected from AI Knowledge Agent’s ai_languageModel output.  
    - Output: Sends generated text back to AI Knowledge Agent.  
    - Edge cases: API authentication failures, rate limiting, model unavailability, or response timeouts.  
    - Version: 1.2  

#### 2.6 Workflow Annotations (Sticky Notes)

- **Overview:**  
  Provide documentation and instructions embedded within the workflow for maintainers and users.

- **Nodes Involved:**  
  - Sticky Note (Lookio tool explanation)  
  - Sticky Note1 (AI model explanation)  
  - Sticky Note2 (Agent explanation)  
  - Sticky Note3 (Overall workflow and setup instructions)

- **Node Details:**

  - **Sticky Note**  
    - Content explains the purpose of the Lookio tool node, reminding users to add API key and assistant ID.  
    - Positioned near Query knowledge base node.  

  - **Sticky Note1**  
    - Describes the AI model node, instructing to connect OpenAI API or switch providers.  
    - Positioned near OpenAI Chat Model node.  

  - **Sticky Note2**  
    - Explains the AI agent’s role and guides on customizing system messages and response style.  
    - Positioned near AI Knowledge Agent node.  

  - **Sticky Note3**  
    - Provides a detailed overview of the entire workflow, setup instructions, and credits Guillaume Duvernay as the author.  
    - Positioned above the workflow’s input nodes for visibility.  

---

### 3. Summary Table

| Node Name               | Node Type                                    | Functional Role                          | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                 |
|-------------------------|----------------------------------------------|----------------------------------------|--------------------------|-------------------------|---------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger        | Receives incoming chat messages        | None                     | AI Knowledge Agent      |                                                                                             |
| AI Knowledge Agent      | @n8n/n8n-nodes-langchain.agent               | Main AI processing and decision making | When chat message received | OpenAI Chat Model, Simple Memory, Query knowledge base | Sticky Note2: Explains agent role and system message configuration                           |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi        | Generates GPT-based chat responses     | AI Knowledge Agent       | AI Knowledge Agent      | Sticky Note1: Core AI model setup and OpenAI API connection instructions                    |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow  | Maintains conversation context         | AI Knowledge Agent       | AI Knowledge Agent      |                                                                                             |
| Query knowledge base    | n8n-nodes-base.httpRequestTool                | Queries Lookio knowledge base API      | AI Knowledge Agent       | AI Knowledge Agent      | Sticky Note: Lookio tool instructions and placeholders for API key and assistant ID         |
| Sticky Note             | n8n-nodes-base.stickyNote                      | Documentation and instructions          | None                     | None                    | See above descriptions                                                                       |
| Sticky Note1            | n8n-nodes-base.stickyNote                      | Documentation and instructions          | None                     | None                    | See above descriptions                                                                       |
| Sticky Note2            | n8n-nodes-base.stickyNote                      | Documentation and instructions          | None                     | None                    | See above descriptions                                                                       |
| Sticky Note3            | n8n-nodes-base.stickyNote                      | Documentation and instructions          | None                     | None                    | Overview, setup instructions, and credits                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add “When chat message received” node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook (auto-generated or custom).  
   - No additional options necessary.  

3. **Add “AI Knowledge Agent” node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure the system message with the following text (multi-line):  
     ```
     You are a helpful assistant that answers the user based on a knowledge base.

     Whenever the user query requires specific knowledge (most queries except empty queries like "hi"), call the tool "Query knowledge base" with a question to have it output an answer based on the knowledge base.

     If the output from the knowledge base tool indicates that the knowledge base doesn't contain enough insights to answer, communicate this to the user transparently.
     ```  
   - Add a tool named “Query knowledge base” referencing the HTTP Request node you will create next.  

4. **Add “Query knowledge base” node:**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - Configure HTTP Request:  
     - URL: `https://api.lookio.app/webhook/query`  
     - Method: POST  
     - Body parameters:  
       - `query`: expression to dynamically get the question from AI agent input:  
         ```
         {{$fromAI('parameters0_Value', `The query to the knowledge base, in the form of a question`, 'string')}}
         ```  
       - `assistant_id`: your Lookio assistant ID (replace placeholder).  
       - `query_mode`: "flash"  
     - Headers:  
       - `api_key`: your Lookio API key (replace placeholder).  
     - Enable sending body and headers.  
   - Set tool description to clarify usage for knowledge base querying.  

5. **Add “OpenAI Chat Model” node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Select model: `gpt-4.1-mini`  
   - Connect your OpenAI API credentials.  

6. **Add “Simple Memory” node:**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Use default settings for buffer window size and behavior.  

7. **Connect nodes:**  
   - Connect the output of “When chat message received” to the input of “AI Knowledge Agent.”  
   - Connect “AI Knowledge Agent” to “OpenAI Chat Model” through the `ai_languageModel` output.  
   - Connect “AI Knowledge Agent” to “Simple Memory” through the `ai_memory` output and back.  
   - Connect “AI Knowledge Agent” to “Query knowledge base” through the `ai_tool` output.  
   - Ensure all outputs of the AI Knowledge Agent nodes are correctly connected to their respective nodes as per the above connections.  

8. **Add Sticky Notes (optional but recommended):**  
   - Add notes near nodes with content describing the role and configuration, as per the original workflow for clarity.  

9. **Replace placeholders:**  
   - In “Query knowledge base” node, replace `<your-lookio-api-key>` with your actual Lookio API key.  
   - Replace `<your-assistant-id>` with your Lookio assistant ID.  
   - Ensure OpenAI credentials are correctly configured in the “OpenAI Chat Model” node.  

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                    |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Lookio is a platform to build company knowledge bases easily, supporting intelligent Q&A bots.                                     | https://www.lookio.app/                            |
| The AI agent minimizes API usage by handling simple greetings without querying the knowledge base, saving credits.                  | Sticky Note3 content                               |
| The workflow template was developed by Guillaume Duvernay, a recognized expert in AI workflows.                                    | Sticky Note3 content                               |
| For more information on configuring OpenAI nodes in n8n, refer to official docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-openai/ | n8n Official Documentation                         |
| Remember to secure your API keys and never expose them publicly.                                                                    | General best practice                              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---