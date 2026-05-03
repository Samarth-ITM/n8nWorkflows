Question and Answer AI Agent Chatbot [2/2]

https://n8nworkflows.xyz/workflows/question-and-answer-ai-agent-chatbot--2-2--13354


# Question and Answer AI Agent Chatbot [2/2]

## 1. Workflow Overview

**Purpose:** Provide an end-user chat Q&A experience powered by an AI Agent that **grounds answers** in a stored knowledge base of **Question/Answer pairs** located in an **n8n Data Table**.

**Typical use cases:**
- Internal helpdesk / FAQ bot backed by curated Q&A rows
- Product support chat where answers must come from an approved knowledge base
- Lightweight “RAG-like” chatbot without an external vector database

**Logical blocks (by dependencies):**
1.1 **Chat Entry Point** → receives a chat message from an n8n Chat UI/session.  
1.2 **Agent Orchestration (LLM + Memory + Tooling)** → AI Agent uses OpenAI chat model, keeps short conversation history, and calls a Data Table Tool to search Q&A rows.  
1.3 **Knowledge Base Retrieval (Data Table Tool)** → queries the Data Table using partial matching against *Question* and *Tags*, returns matching rows for grounding.

---

## 2. Block-by-Block Analysis

### 2.1 Chat Entry Point

**Overview:** Starts the workflow when a user sends a message in the n8n chat interface. Passes the message to the AI Agent.

**Nodes involved:**
- When chat message received

#### Node: **When chat message received**
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — chat trigger / workflow entrypoint for n8n’s chat.
- **Key configuration (interpreted):**
  - **Agent name in chat:** “QA Chatbot” (used for chat UI identity/selection).
  - **Available in chat:** enabled (this workflow appears as a chat agent).
- **Inputs / outputs:**
  - **Output:** Main output → **AI Agent**
- **Version notes:** TypeVersion **1.4**.
- **Edge cases / failures:**
  - If the chat feature/UI is not enabled or reachable, the trigger won’t fire.
  - If the workflow is inactive, no chat sessions will route here.

---

### 2.2 Agent Orchestration (LLM + Memory + Tool)

**Overview:** The AI Agent receives the user message, maintains short-term memory, and uses a dedicated tool to retrieve relevant Q&A rows from the Data Table. It then answers strictly based on the retrieved context, or declines when nothing relevant exists.

**Nodes involved:**
- AI Agent
- OpenAI Chat Model
- Simple Memory
- fetch-qa-from-db

#### Node: **AI Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates the conversation, decides whether to call tools, and generates final responses.
- **Key configuration (interpreted):**
  - **System message (behavior policy):**
    - Answer questions **based on feedback data** (the Q&A table).
    - **Use tools** to search by querying **‘question’ and ‘tags’** columns.
    - If multiple entries match, **synthesize**.
    - If nothing relevant is found, **say you don’t have that info** in the QA database.
- **Key expressions / variables:** None directly inside the agent; tool inputs are generated via `$fromAI(...)` inside the tool node.
- **Inputs / outputs (connections):**
  - **Main input:** from **When chat message received**
  - **AI language model input:** from **OpenAI Chat Model**
  - **AI memory input:** from **Simple Memory**
  - **AI tool input:** from **fetch-qa-from-db**
  - **Main output:** back to chat (handled implicitly by the Chat Trigger/Agent integration)
- **Version notes:** TypeVersion **3.1**.
- **Edge cases / failures:**
  - Model/tool mismatch: if the LLM cannot call tools (or tool calling is limited), the agent may answer without grounding.
  - Prompt adherence risk: user prompts may try to override grounding; system message mitigates but does not guarantee perfect compliance.
  - Empty tool results: agent should follow system message and decline politely; if not, you may need stricter instructions.

#### Node: **OpenAI Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat completion model used by the agent.
- **Key configuration (interpreted):**
  - **Model:** `gpt-5-mini`
  - **Built-in tools:** none enabled here (agent uses the Data Table Tool node instead).
- **Credentials:**
  - Uses OpenAI credential: **“n8n free OpenAI API credits”**
- **Inputs / outputs:**
  - Connected as **AI languageModel** → into **AI Agent**
- **Version notes:** TypeVersion **1.3**.
- **Edge cases / failures:**
  - Authentication/quota errors (invalid key, expired credits, rate limiting).
  - Model availability changes (if `gpt-5-mini` is not available on the account/region).
  - Latency/timeouts for long conversations (less likely here, but possible).

#### Node: **Simple Memory**
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversational memory.
- **Key configuration (interpreted):**
  - Default buffer-window behavior (no custom parameters set). Typically keeps a limited recent history window (implementation-defined by node defaults).
- **Inputs / outputs:**
  - Connected as **AI memory** → into **AI Agent**
- **Version notes:** TypeVersion **1.3**.
- **Edge cases / failures:**
  - Memory window may be too small for multi-turn disambiguation; agent might “forget” earlier constraints.
  - If chat sessions are not properly separated by the platform, memory might not behave as expected (depends on n8n chat session handling).

#### Node: **fetch-qa-from-db**
- **Type / role:** `n8n-nodes-base.dataTableTool` — exposes an n8n Data Table query as an **LLM-callable tool**.
- **Key configuration (interpreted):**
  - **Operation:** Get rows
  - **Return:** Return all matches (`returnAll: true`)
  - **Target Data Table:** Data table named **“q&a”** (ID is selected from the list)
  - **Filters:** Two conditions are configured:
    1. **Question** `ilike` *search term*  
       - Value is provided dynamically by the agent via:
         - `{{$fromAI('conditions0_Value', 'The search term or question to look for...', 'string')}}`
       - Intended usage: single words or short fragments; matches questions containing the fragment (case-insensitive).
    2. **Tags** `ilike` *tag term*  
       - Value is provided dynamically by the agent via:
         - `{{$fromAI('conditions1_Value', 'key terms for this question:answer pair...', 'string')}}`
       - Intended usage: one search term per tool invocation.
  - **Tool description (what the agent “sees”):**
    - Search Feedback data table; match against ‘question’; returns entries with answers; provide specific search terms.
- **Inputs / outputs:**
  - Connected as **AI tool** → into **AI Agent**
  - Tool is invoked by the agent when it decides retrieval is necessary.
- **Version notes:** TypeVersion **1.1**.
- **Edge cases / failures:**
  - **Schema mismatch:** If the Data Table columns are not exactly **“Question”** and **“Tags”** (capitalization matters), the query may return nothing or error.
  - **Over-broad matching:** `ilike` with short fragments can return many rows; agent may need to refine queries.
  - **Under-matching:** If tags are sparse or inconsistent, `Tags ilike` may miss relevant entries.
  - **Access issues:** Missing permissions to the Data Table or project-level access changes.

---

### 2.3 Documentation / Annotation Block (Sticky Note)

**Overview:** Provides human context: identifies this as Flow 2/2 and credits the author.

**Nodes involved:**
- Sticky Note

#### Node: **Sticky Note**
- **Type / role:** `n8n-nodes-base.stickyNote` — annotation only.
- **Content (preserved):**
  - “## QA Ingest Template  
    ### Flow 2/2  
    This workflow provides an AI Agent that answers user questions by retrieving relevant Q&A pairs from an n8n Data Table as grounding context, ensuring responses are informed by the stored knowledge base.  
    By [Max Tkacz | The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/)”
- **Version notes:** TypeVersion **1**.
- **Edge cases / failures:** None (non-executable).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry trigger | — | AI Agent | ## QA Ingest Template / Flow 2/2 — AI Agent answers user questions using Q&A pairs from an n8n Data Table. By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates LLM + memory + retrieval tool; generates grounded answers | When chat message received; OpenAI Chat Model (ai_languageModel); Simple Memory (ai_memory); fetch-qa-from-db (ai_tool) | (Chat response via trigger integration) | ## QA Ingest Template / Flow 2/2 — AI Agent answers user questions using Q&A pairs from an n8n Data Table. By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM powering the agent | — | AI Agent | ## QA Ingest Template / Flow 2/2 — AI Agent answers user questions using Q&A pairs from an n8n Data Table. By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores recent chat turns for context | — | AI Agent | ## QA Ingest Template / Flow 2/2 — AI Agent answers user questions using Q&A pairs from an n8n Data Table. By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| fetch-qa-from-db | n8n-nodes-base.dataTableTool | LLM tool to query Q&A rows from Data Table by Question/Tags | — (invoked by agent as tool) | AI Agent | ## QA Ingest Template / Flow 2/2 — AI Agent answers user questions using Q&A pairs from an n8n Data Table. By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |
| Sticky Note | n8n-nodes-base.stickyNote | Annotation / credits | — | — | ## QA Ingest Template / Flow 2/2 — AI Agent answers user questions using Q&A pairs from an n8n Data Table. By [Max Tkacz \| The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create/verify the Data Table (knowledge base)**
   1. In n8n, go to **Data Tables** and create (or use) a table named **q&a**.
   2. Ensure it has at least these columns (exact names recommended):
      - **Question** (text)
      - **Answer** (text) *(not explicitly referenced in filters, but expected to exist for useful output)*
      - **Tags** (text)
   3. Populate rows with Q&A pairs and tags.

2) **Add the Chat Trigger**
   1. Create node: **When chat message received**
      - Type: **Chat Trigger** (`@n8n/n8n-nodes-langchain.chatTrigger`)
      - Set:
        - **Agent Name:** `QA Chatbot`
        - **Available in Chat:** enabled

3) **Add the AI Agent**
   1. Create node: **AI Agent**
      - Type: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
      - Set **System Message** to:
        - “You are a helpful Q&A assistant that answers questions based on feedback data. Use tools to search for relevant information by querying the 'question' and 'tags' columns. When a user asks a question, search the QA database to find matching questions or relevant tags, then provide the corresponding answer. If you find multiple relevant entries, synthesize the information. If no relevant information is found, politely inform the user that you don't have that information in the QA database.”
   2. Connect **When chat message received → AI Agent** (main connection).

4) **Add the OpenAI Chat Model**
   1. Create node: **OpenAI Chat Model**
      - Type: **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
      - Choose model: `gpt-5-mini`
   2. **Credentials:** configure an OpenAI API credential (API key / org as required by your n8n setup).
   3. Connect **OpenAI Chat Model → AI Agent** using the **ai_languageModel** connection type.

5) **Add Memory**
   1. Create node: **Simple Memory**
      - Type: **Simple Memory / Buffer Window** (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)
      - Leave defaults unless you need a larger/smaller memory window.
   2. Connect **Simple Memory → AI Agent** using the **ai_memory** connection type.

6) **Add the Data Table Tool**
   1. Create node: **fetch-qa-from-db**
      - Type: **Data Table Tool** (`n8n-nodes-base.dataTableTool`)
      - Configure:
        - **Data Table:** select your **q&a** table
        - **Operation:** `Get`
        - **Return All:** enabled
        - **Filters:** add two conditions:
          - Condition A:
            - Column: **Question**
            - Operator: **ilike**
            - Value: use expression with `$fromAI(...)` (AI-provided tool input)
          - Condition B:
            - Column: **Tags**
            - Operator: **ilike**
            - Value: use expression with `$fromAI(...)`
        - **Tool Description:** explain that this searches the Q&A/feedback table and returns matching entries (so the agent knows when/how to call it).
   2. Connect **fetch-qa-from-db → AI Agent** using the **ai_tool** connection type.

7) **(Optional) Add the Sticky Note**
   1. Add a **Sticky Note** and paste:
      - “## QA Ingest Template  
        ### Flow 2/2  
        This workflow provides an AI Agent that answers user questions by retrieving relevant Q&A pairs from an n8n Data Table as grounding context, ensuring responses are informed by the stored knowledge base.  
        By [Max Tkacz | The Original Flowgrammer](https://www.linkedin.com/in/maxtkacz/)”

8) **Activate and test**
   1. Activate the workflow.
   2. Open the n8n chat interface and ask a question that matches a **Question** fragment or a **Tags** term in the table.
   3. Confirm the agent calls the tool and answers using returned rows; test a non-covered question to ensure it declines per system message.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This template is part of the official n8n quick start (2026). Watch it here: https://www.youtube.com/watch?v=GuaKeDS6UKU | Provided in workflow description |
| By Max Tkacz \| The Original Flowgrammer | https://www.linkedin.com/in/maxtkacz/ |
| “QA Ingest Template — Flow 2/2” (this workflow is the answering/serving side using a Data Table as grounding context) | Sticky note context |