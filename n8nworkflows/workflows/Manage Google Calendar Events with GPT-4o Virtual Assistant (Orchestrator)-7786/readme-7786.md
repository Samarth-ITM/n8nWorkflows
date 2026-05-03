Manage Google Calendar Events with GPT-4o Virtual Assistant (Orchestrator)

https://n8nworkflows.xyz/workflows/manage-google-calendar-events-with-gpt-4o-virtual-assistant--orchestrator--7786


# Manage Google Calendar Events with GPT-4o Virtual Assistant (Orchestrator)

### 1. Workflow Overview

This workflow, titled **"Manage Google Calendar Events with GPT-4o Virtual Assistant (Orchestrator)"**, acts as a conversational orchestrator that integrates a chat interface with Google Calendar management capabilities via an AI assistant named Albert. It receives chat messages, processes them through a GPT-4o-powered AI agent, and delegates calendar-related tasks (get, create, delete events) to a sub-workflow specialized in Google Calendar operations.

**Target Use Cases:**  
- Conversational assistants managing calendar events through natural language.  
- Automating calendar event queries, creation, and deletion seamlessly via chat inputs.  
- Maintaining conversational context using memory buffers to handle multi-turn interactions.

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving chat messages from users via webhook trigger.  
- **1.2 AI Processing and Orchestration:** Processing input with GPT-4o model, managing conversation memory, and orchestrating task delegation.  
- **1.3 Calendar Task Execution:** Invoking a sub-workflow dedicated to Google Calendar event management.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming chat messages via a webhook and initializes the conversation with the AI assistant.

**Nodes Involved:**  
- When chat message received  
- Sticky Note (contextual information)

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (LangChain)  
  - *Role:* Entry point; triggers workflow upon receiving chat messages.  
  - *Configuration:* Uses a webhook ID for external chat message delivery. No additional options configured.  
  - *Inputs:* External chat messages containing user queries.  
  - *Outputs:* Passes chat message JSON including `chatInput` and `sessionId` to next node.  
  - *Failure modes:* Webhook connectivity issues, malformed chat input.  
  - *Version:* 1.3.

- **Sticky Note**  
  - *Type:* Sticky Note (n8n UI element)  
  - *Role:* Documentation; explains the trigger and assistant persona.  
  - *Content:*  
    ```
    🟢 PARENT — CHAT ENTRY

    Trigger: chat message
    Persona: Albert (calendar assistant)
    Uses tool: sub_agent_cal (Execute Workflow Tool)

    chatInput ➜ text, sessionId ➜ sessionid
    ```

---

#### 1.2 AI Processing and Orchestration

**Overview:**  
This block processes the input chat message through the GPT-4o language model, maintains conversational context via a memory buffer, and orchestrates execution of calendar tasks by invoking the sub-agent workflow.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory  
- Sticky Note1 (documentation)

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Central orchestrator that interprets input, decides actions, and delegates tasks.  
  - *Configuration:*  
    - System message defines persona as "Albert, a helpful google calendar assistant."  
    - Uses sub-agent workflow `sub_agent_cal` to perform actual calendar CRUD operations.  
  - *Inputs:* Receives chat message from trigger, memory state, and language model outputs.  
  - *Outputs:* Sends commands to sub-agent, updates memory, responds back via chat.  
  - *Failure modes:* Expression errors in system message, execution failure of sub-agent, timeouts.  
  - *Version:* 2.2.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides natural language understanding and generation with GPT-4o-mini model.  
  - *Configuration:*  
    - Model set to `gpt-4o-mini`.  
    - Uses OpenAI credentials stored as `OpenAi account`.  
  - *Inputs:* Receives prompt from AI Agent.  
  - *Outputs:* Returns language model completions to AI Agent.  
  - *Failure modes:* API authentication issues, rate limits, model unavailability.  
  - *Version:* 1.2.

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains conversational context over recent interactions to enable multi-turn dialogue.  
  - *Configuration:* Default buffer window.  
  - *Inputs:* Conversation data from AI Agent.  
  - *Outputs:* Updated memory context to AI Agent.  
  - *Failure modes:* Memory overflow or corruption, state mismatch.  
  - *Version:* 1.3.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Documentation; explains orchestration design.  
  - *Content:*  
    ```
    🔗 ORCHESTRATION

    Parent = intent + memory
    Child = tools (Get / Create / Delete)
    ```

---

#### 1.3 Calendar Task Execution (Sub-Workflow Invocation)

**Overview:**  
This block delegates calendar event management tasks to a specialized sub-workflow which handles Google Calendar API interactions such as fetching, creating, or deleting events.

**Nodes Involved:**  
- sub_agent_cal

**Node Details:**

- **sub_agent_cal**  
  - *Type:* LangChain Tool Workflow (Execute Workflow Node)  
  - *Role:* Acts as a tool invoked by the AI Agent to execute calendar operations.  
  - *Configuration:*  
    - Points to sub-workflow ID `41tGOnU3hN9zjJln` named "Google_Cal_sub_agent".  
    - Passes inputs:  
      - `text` from chat message (`$json.chatInput`)  
      - `session_id` from chat session ID (`$json.sessionId`)  
  - *Inputs:* Receives delegated commands and parameters from AI Agent.  
  - *Outputs:* Returns results of calendar operations back to AI Agent.  
  - *Failure modes:* Sub-workflow unavailability, API auth failures, invalid inputs, Google API rate limits.  
  - *Version:* 2.2.  
  - *Note:* This node is crucial for actual calendar CRUD, encapsulated in the sub-workflow.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                          | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                           |
|-------------------------|-------------------------------------|----------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger              | Entry trigger receiving chat messages  | -                           | AI Agent                   | 🟢 PARENT — CHAT ENTRY<br>Trigger: chat message<br>Persona: Albert (calendar assistant)<br>Uses tool: sub_agent_cal    |
| AI Agent                | LangChain Agent                     | Orchestrator managing AI logic and tools | When chat message received, OpenAI Chat Model, Simple Memory, sub_agent_cal | sub_agent_cal (tool), Simple Memory | 🔗 ORCHESTRATION<br>Parent = intent + memory<br>Child = tools (Get / Create / Delete)                                   |
| OpenAI Chat Model       | LangChain OpenAI Chat Model         | Language model for NLP processing      | AI Agent                    | AI Agent                   |                                                                                                                       |
| Simple Memory           | LangChain Memory Buffer Window      | Maintains conversational context      | AI Agent                    | AI Agent                   |                                                                                                                       |
| sub_agent_cal           | LangChain Tool Workflow             | Executes Google Calendar event tasks   | AI Agent                    | AI Agent                   |                                                                                                                       |
| Sticky Note             | n8n Sticky Note                    | Documentation for Input Reception      | -                           | -                          | 🟢 PARENT — CHAT ENTRY<br>Trigger: chat message<br>Persona: Albert (calendar assistant)<br>Uses tool: sub_agent_cal    |
| Sticky Note1            | n8n Sticky Note                    | Documentation for Orchestration Logic  | -                           | -                          | 🔗 ORCHESTRATION<br>Parent = intent + memory<br>Child = tools (Get / Create / Delete)                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name it "When chat message received".  
   - Configure webhook to receive chat messages externally.  
   - No special options required.  

2. **Create AI Agent Node**  
   - Add node: `@n8n/n8n-nodes-langchain.agent`  
   - Name it "AI Agent".  
   - Set system message to:  
     `"You are Albert, a helpful google calendar assistant. You are to use the \"sub_agent_cal\" to get calendar events, create calendar events, and delete calendar events."`  
   - No additional options needed.  

3. **Create OpenAI Chat Model Node**  
   - Add node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name it "OpenAI Chat Model".  
   - Set model to `gpt-4o-mini`.  
   - Assign OpenAI credentials with valid API key (named "OpenAi account" or equivalent).  

4. **Create Simple Memory Node**  
   - Add node: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Name it "Simple Memory".  
   - Use default settings (buffer window).  

5. **Create Sub-Workflow Execution Node**  
   - Add node: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Name it "sub_agent_cal".  
   - Configure to execute sub-workflow with ID matching your Google Calendar sub-agent workflow (e.g., `41tGOnU3hN9zjJln`).  
   - Map inputs:  
     - `text` ← `{{$json.chatInput}}`  
     - `session_id` ← `{{$json.sessionId}}`  

6. **Connect Nodes in Order:**  
   - "When chat message received" → "AI Agent" (main)  
   - "OpenAI Chat Model" → "AI Agent" (ai_languageModel)  
   - "Simple Memory" → "AI Agent" (ai_memory)  
   - "sub_agent_cal" → "AI Agent" (ai_tool)  

7. **Set Credentials:**  
   - Assign OpenAI API credentials in "OpenAI Chat Model".  
   - Ensure the sub-workflow for Google Calendar operations has proper Google OAuth2 credentials configured.  

8. **Add Sticky Notes (Optional but Recommended):**  
   - Add sticky notes with the provided content to document trigger and orchestration logic for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The AI assistant persona is named "Albert" and acts as a Google Calendar virtual assistant.   | Workflow system message and design.                                                                             |
| The sub-agent workflow `Google_Cal_sub_agent` is separately responsible for Google Calendar API calls including get, create, and delete event operations. | Sub-workflow reference ID `41tGOnU3hN9zjJln`.                                                                   |
| OpenAI GPT-4o-mini is used for natural language understanding and generation within the AI Agent. | Model selection in OpenAI Chat Model node.                                                                      |
| For detailed Google Calendar API integration, ensure proper OAuth2 authentication in the sub-workflow. | Best practice for Google Calendar API usage.                                                                    |
| The workflow uses LangChain nodes for advanced orchestration of AI tools and memory management. | Documentation: https://docs.n8n.io/integrations/automation/langchain/                                           |

---

*Disclaimer:* The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.