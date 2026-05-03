🧠 Empower Your AI Chatbot with Long-Term Memory and Dynamic Tool Routing

https://n8nworkflows.xyz/workflows/---empower-your-ai-chatbot-with-long-term-memory-and-dynamic-tool-routing-3025


# 🧠 Empower Your AI Chatbot with Long-Term Memory and Dynamic Tool Routing

### 1. Workflow Overview

This workflow, titled **"Empower Your AI Chatbot with Long-Term Memory and Dynamic Tool Routing"**, is designed to enhance an AI chatbot with persistent long-term memory and dynamic task routing capabilities. It targets AI developers, businesses, customer support teams, marketers, and researchers who want to build intelligent, context-aware chatbots that can save and retrieve memories, route tasks dynamically to various tools, and send notifications via multiple channels.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures incoming chat messages or external workflow triggers.
- **1.2 AI Processing with Short-Term Memory**: Uses OpenAI GPT-based models and a window buffer memory for immediate context understanding.
- **1.3 Dynamic Tools Router**: Routes tasks based on AI instructions to appropriate tools such as saving memories, retrieving memories, or sending notifications.
- **1.4 Long-Term Memory Management**: Implements saving and retrieving memories using Google Docs as persistent storage.
- **1.5 Notification Dispatch**: Sends memory summaries or updates via Telegram and Gmail.
- **1.6 Message Preparation**: Formats content appropriately for Telegram or Gmail before sending.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives input either from an external workflow trigger or a chat message trigger, initiating the AI processing pipeline.

- **Nodes Involved:**  
  - `When Executed by Another Workflow`  
  - `Ⓜ️ When chat message received`

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point when this workflow is called by another workflow, passing input data through.  
    - Configuration: Pass-through input source.  
    - Connections: Outputs to `Memory Tool Router`.  
    - Edge Cases: Input data format mismatch may cause routing errors.

  - **Ⓜ️ When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Entry point for chat messages from users.  
    - Configuration: Default options, webhook ID assigned.  
    - Connections: Outputs to `🧠 AI Agent w/Long Term Memory`.  
    - Edge Cases: Webhook connectivity issues, message format errors.

---

#### 1.2 AI Processing with Short-Term Memory

- **Overview:**  
  Processes incoming messages using an OpenAI GPT model combined with a window buffer memory to maintain short-term conversational context.

- **Nodes Involved:**  
  - `🤖OpenAI Chat Model`  
  - `🤯Window Buffer Memory`  
  - `🧠 AI Agent w/Long Term Memory`

- **Node Details:**

  - **🤖OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides AI language model processing using GPT-4o-mini.  
    - Configuration: Model set to "gpt-4o-mini", no additional options.  
    - Connections: Outputs to `🧠 AI Agent w/Long Term Memory`.  
    - Credentials: OpenAI API credentials required.  
    - Edge Cases: API rate limits, authentication failures.

  - **🤯Window Buffer Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a sliding window of the last 10 conversational exchanges for short-term memory.  
    - Configuration: Context window length set to 10.  
    - Connections: Feeds memory context into `🧠 AI Agent w/Long Term Memory`.  
    - Edge Cases: Memory overflow if window size is too small for conversation length.

  - **🧠 AI Agent w/Long Term Memory**  
    - Type: LangChain Agent  
    - Role: Central AI agent that integrates short-term memory, long-term memory tools, and task routing.  
    - Configuration:  
      - System message defines agent behavior, listing available tools and rules.  
      - Tools include saving/retrieving memories and sending notifications.  
      - Current date/time dynamically inserted.  
    - Connections: Receives input from chat trigger, OpenAI model, and window buffer memory; outputs to `🧠Save Memories` tool or others via routing.  
    - Credentials: OpenAI API required.  
    - Edge Cases: Misinterpretation of tool instructions, API errors, unexpected input formats.

---

#### 1.3 Dynamic Tools Router

- **Overview:**  
  Routes AI agent instructions to the appropriate tool workflow based on the `route` field in the input JSON.

- **Nodes Involved:**  
  - `Memory Tool Router`

- **Node Details:**

  - **Memory Tool Router**  
    - Type: Switch Node  
    - Role: Evaluates the `route` property in incoming data to direct flow to one of four outputs: Save, Retrieve, Telegram, or Gmail.  
    - Configuration:  
      - Routes:  
        - `"save_memory"` → Save Memories  
        - `"retrieve_memory"` → Retrieve Memories  
        - `"send_memories_to_telegram"` → Telegram Notification  
        - `"send_memories_to_gmail"` → Gmail Notification  
    - Connections: Outputs to multiple Google Docs nodes or tool workflows accordingly.  
    - Edge Cases: Missing or invalid `route` values cause no routing; case sensitivity enforced.

---

#### 1.4 Long-Term Memory Management

- **Overview:**  
  Implements saving and retrieving long-term memories using Google Docs as persistent storage.

- **Nodes Involved:**  
  - `Save Long Term Memories`  
  - `Saved response`  
  - `Retrieve Long Term Memories`  
  - `Respond with long term memories`  
  - `Retrieve Long Term Memories2`  
  - `Retrieve Long Term Memories3`

- **Node Details:**

  - **Save Long Term Memories**  
    - Type: Google Docs Node  
    - Role: Inserts new memory entries into a Google Docs document.  
    - Configuration:  
      - Operation: Update (insert)  
      - Document URL: Fixed Google Docs document ID  
      - Inserted JSON includes current date/time and memory content from `$json.query`.  
    - Credentials: Google Docs OAuth2 required.  
    - Connections: Outputs to `Saved response`.  
    - Edge Cases: Google Docs API errors, permission issues, document URL invalid.

  - **Saved response**  
    - Type: Set Node  
    - Role: Sets a confirmation response `"Memory saved"`.  
    - Configuration: Assigns string `"Memory saved"` to `response` field.  
    - Connections: Terminal node after saving memory.  
    - Edge Cases: None significant.

  - **Retrieve Long Term Memories** (and variants 2 & 3)  
    - Type: Google Docs Node  
    - Role: Retrieves content from the same Google Docs document.  
    - Configuration:  
      - Operation: Get  
      - Document URL: Same fixed Google Docs document ID  
    - Credentials: Google Docs OAuth2 required.  
    - Connections:  
      - `Retrieve Long Term Memories` → `Respond with long term memories`  
      - `Retrieve Long Term Memories2` → `Prepare Telegram Message`  
      - `Retrieve Long Term Memories3` → `Prepare Gmail Message`  
    - Edge Cases: API errors, document access issues.

  - **Respond with long term memories**  
    - Type: Set Node  
    - Role: Sets the retrieved document content as the response string.  
    - Configuration: Assigns `{{$json.content}}` to `response`.  
    - Connections: Terminal node for retrieval response.  
    - Edge Cases: Empty or malformed document content.

---

#### 1.5 Notification Dispatch

- **Overview:**  
  Sends formatted memory summaries or updates via Telegram or Gmail.

- **Nodes Involved:**  
  - `Send Success Message to Telegram`  
  - `Email Workflow Stats`

- **Node Details:**

  - **Send Success Message to Telegram**  
    - Type: Telegram Node  
    - Role: Sends a message to a Telegram chat with memory content.  
    - Configuration:  
      - Text: Prefixed with `"n8n User Memories\n"` plus dynamic content from `$json.text`.  
      - Chat ID: From environment variable `TELEGRAM_CHAT_ID`.  
      - Parse Mode: HTML  
      - Attribution disabled.  
    - Credentials: Telegram API OAuth2 required.  
    - Edge Cases: Telegram API errors, invalid chat ID.

  - **Email Workflow Stats**  
    - Type: Gmail Node  
    - Role: Sends an email with memory content.  
    - Configuration:  
      - Recipient: From environment variable `EMAIL_ADDRESS_JOE`.  
      - Subject: `"n8n User Memories"`  
      - Message body: From `$json.text` (HTML formatted).  
      - Attribution disabled.  
    - Credentials: Gmail OAuth2 required.  
    - Edge Cases: Gmail API errors, invalid recipient address.

---

#### 1.6 Message Preparation

- **Overview:**  
  Formats retrieved memory content into appropriate formats for Telegram (simple list) or Gmail (HTML table).

- **Nodes Involved:**  
  - `Prepare Telegram Message`  
  - `Prepare Gmail Message`

- **Node Details:**

  - **Prepare Telegram Message**  
    - Type: LangChain Chain LLM Node  
    - Role: Formats content into a simple, unformatted list suitable for Telegram messages.  
    - Configuration: Prompt instructs to avoid preamble or explanation, output plain list.  
    - Connections: Outputs to `Send Success Message to Telegram`.  
    - Edge Cases: Formatting errors, unexpected content structure.

  - **Prepare Gmail Message**  
    - Type: LangChain Chain LLM Node  
    - Role: Formats content into a modern HTML table (max 800px width) for email body.  
    - Configuration: Prompt instructs to avoid preamble, remove markdown code blocks.  
    - Connections: Outputs to `Email Workflow Stats`.  
    - Edge Cases: HTML formatting issues, content too large.

---

#### Additional Tool Workflows (Tool Nodes)

- **Overview:**  
  These nodes represent modular tool workflows callable by the AI agent for specific tasks.

- **Nodes Involved:**  
  - `🧠Save Memories` (save_long_term_memory)  
  - `🔎Retrieve Memories` (retrieve_long_term_memory)  
  - `📤Send Memories to Telegram` (send_memories_to_telegram)  
  - `📫Send Memories to Gmail` (send_memories_to_gmail)

- **Node Details:**

  - Each is a LangChain Tool Workflow node configured with:  
    - A unique name matching the tool function.  
    - A `route` field set to the corresponding route string.  
    - Workflow ID set to this workflow's ID (self-reference).  
    - Description explaining the tool's purpose.  
  - These nodes enable the AI agent to invoke specific sub-workflows dynamically.  
  - Edge Cases: Recursive calls must be managed carefully to avoid infinite loops.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                          | Input Node(s)                    | Output Node(s)                    | Sticky Note                                                                                      |
|-------------------------------|--------------------------------|----------------------------------------|---------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger       | Entry point for external workflow calls | -                               | Memory Tool Router               |                                                                                                 |
| Ⓜ️ When chat message received   | Chat Trigger (LangChain)        | Entry point for chat messages           | -                               | 🧠 AI Agent w/Long Term Memory   |                                                                                                 |
| 🤖OpenAI Chat Model             | LangChain OpenAI Chat Model     | AI language model processing             | Ⓜ️ When chat message received    | 🧠 AI Agent w/Long Term Memory   | ## Cloud LLM                                                                                     |
| 🤯Window Buffer Memory          | LangChain Memory Buffer Window  | Maintains short-term chat memory         | -                               | 🧠 AI Agent w/Long Term Memory   | ## Short Term Chat Memory                                                                        |
| 🧠 AI Agent w/Long Term Memory  | LangChain Agent                 | Central AI agent integrating memory & tools | 🤖OpenAI Chat Model, 🤯Window Buffer Memory, Ⓜ️ When chat message received | 🧠Save Memories, 🧠Save Memories tool, etc. | # 🧠 AI Agent Chatbot With Long Term Memory Tools                                                |
| Memory Tool Router              | Switch Node                    | Routes tasks to appropriate tools        | When Executed by Another Workflow | Save Long Term Memories, Retrieve Long Term Memories, Retrieve Long Term Memories2, Retrieve Long Term Memories3 | ## Memory Tool Router                                                                           |
| Save Long Term Memories         | Google Docs Node               | Saves memories to Google Docs             | Memory Tool Router              | Saved response                   | ## 1️⃣ Save Memories                                                                            |
| Saved response                 | Set Node                      | Confirms memory saved                     | Save Long Term Memories         | -                              |                                                                                                 |
| Retrieve Long Term Memories     | Google Docs Node               | Retrieves memories from Google Docs       | Memory Tool Router              | Respond with long term memories  | ## 2️⃣ Retrieve Memories                                                                        |
| Respond with long term memories | Set Node                      | Sets retrieved memories as response       | Retrieve Long Term Memories     | -                              |                                                                                                 |
| Retrieve Long Term Memories2    | Google Docs Node               | Retrieves memories (variant for Telegram) | Memory Tool Router              | Prepare Telegram Message         |                                                                                                 |
| Prepare Telegram Message        | LangChain Chain LLM Node       | Formats content for Telegram               | Retrieve Long Term Memories2    | Send Success Message to Telegram | ## 3️⃣ Send Memories to Telegram                                                               |
| Send Success Message to Telegram| Telegram Node                 | Sends message to Telegram chat             | Prepare Telegram Message        | -                              |                                                                                                 |
| Retrieve Long Term Memories3    | Google Docs Node               | Retrieves memories (variant for Gmail)    | Memory Tool Router              | Prepare Gmail Message            |                                                                                                 |
| Prepare Gmail Message           | LangChain Chain LLM Node       | Formats content for Gmail email             | Retrieve Long Term Memories3    | Email Workflow Stats            | ## 4️⃣ Send Memories to Gmail                                                                  |
| Email Workflow Stats            | Gmail Node                    | Sends email with memory content            | Prepare Gmail Message           | -                              |                                                                                                 |
| 🧠Save Memories (tool)           | LangChain Tool Workflow Node   | Tool to save long-term memories            | 🧠 AI Agent w/Long Term Memory  | -                              | ## 1️⃣ Save Long Term Memories Tool                                                            |
| 🔎Retrieve Memories (tool)       | LangChain Tool Workflow Node   | Tool to retrieve long-term memories        | 🧠 AI Agent w/Long Term Memory  | -                              | ## 2️⃣ Retrieve Long Term Memories Tool                                                        |
| 📤Send Memories to Telegram (tool) | LangChain Tool Workflow Node   | Tool to send memories to Telegram           | 🧠 AI Agent w/Long Term Memory  | -                              | ## 3️⃣ Send Memories to Telegram Tool                                                          |
| 📫Send Memories to Gmail (tool)  | LangChain Tool Workflow Node   | Tool to send memories to Gmail               | 🧠 AI Agent w/Long Term Memory  | -                              | ## 4️⃣ Send Memories to Gmail Tool                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Points:**

   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow` with input source set to "passthrough".
   - Add a **Chat Trigger (LangChain)** node named `Ⓜ️ When chat message received` with default options and assign a webhook ID.

2. **Set Up AI Processing:**

   - Add a **LangChain OpenAI Chat Model** node named `🤖OpenAI Chat Model`:
     - Model: `gpt-4o-mini`
     - Connect input from `Ⓜ️ When chat message received`.
     - Configure with OpenAI API credentials.

   - Add a **LangChain Memory Buffer Window** node named `🤯Window Buffer Memory`:
     - Context window length: 10
     - Connect input appropriately to maintain short-term memory.

   - Add a **LangChain Agent** node named `🧠 AI Agent w/Long Term Memory`:
     - System message:  
       ```
       You are a highly capable and intelligent assistant designed to assist users by leveraging tools for long-term memory, contextual understanding, and real-time information retrieval. You have excellent long term memory. The current date and time is {{ $now }}

       ## TOOLS
       -save_long_term_memory
       -retrieve_long_term_memory
       -send_memories_to_gmail
       -send_memories_to_telegram

       ## RULES
       - If you do not know the answer then consider checking the long term memories.
       ```
     - Connect inputs from `🤖OpenAI Chat Model`, `🤯Window Buffer Memory`, and `Ⓜ️ When chat message received`.
     - Configure OpenAI API credentials.

3. **Configure Tools Router:**

   - Add a **Switch** node named `Memory Tool Router`.
   - Define routing rules based on `$json.route` string values:
     - `"save_memory"` → Output 1
     - `"retrieve_memory"` → Output 2
     - `"send_memories_to_telegram"` → Output 3
     - `"send_memories_to_gmail"` → Output 4
   - Connect input from `When Executed by Another Workflow`.

4. **Implement Long-Term Memory Save:**

   - Add a **Google Docs** node named `Save Long Term Memories`:
     - Operation: Update (insert)
     - Document URL: Use your Google Docs document ID for memory storage.
     - Insert JSON:  
       ```json
       {
         "date": "{{ $now }}",
         "memory": "{{ $json.query }}"
       }
       ```
     - Connect input from `Memory Tool Router` output 1.
     - Configure Google Docs OAuth2 credentials.

   - Add a **Set** node named `Saved response`:
     - Assign `response` = `"Memory saved"`.
     - Connect input from `Save Long Term Memories`.

5. **Implement Long-Term Memory Retrieval:**

   - Add three **Google Docs** nodes named `Retrieve Long Term Memories`, `Retrieve Long Term Memories2`, and `Retrieve Long Term Memories3`:
     - Operation: Get
     - Document URL: Same as above.
     - Connect inputs from `Memory Tool Router` outputs 2, 3, and 4 respectively.

   - Add a **Set** node named `Respond with long term memories`:
     - Assign `response` = `{{$json.content}}`.
     - Connect input from `Retrieve Long Term Memories`.

6. **Prepare Messages for Notifications:**

   - Add a **LangChain Chain LLM** node named `Prepare Telegram Message`:
     - Prompt:  
       ```
       Format this content into a simple unformatted list that can be sent to Telegram: {{ $json.content }}. Avoid any preamble or further explanation.
       ```
     - Connect input from `Retrieve Long Term Memories2`.

   - Add a **LangChain Chain LLM** node named `Prepare Gmail Message`:
     - Prompt:  
       ```
       Format this content into a simple and modern HTML table that is max 800px wide that can be used as the content for an email: {{ $json.content }}. Avoid any preamble or further explanation. Remove all ``` and ```html from response.
       ```
     - Connect input from `Retrieve Long Term Memories3`.

7. **Send Notifications:**

   - Add a **Telegram** node named `Send Success Message to Telegram`:
     - Text:  
       ```
       n8n User Memories
       {{ $json.text }}
       ```
     - Chat ID: Use environment variable `TELEGRAM_CHAT_ID`.
     - Parse mode: HTML
     - Connect input from `Prepare Telegram Message`.
     - Configure Telegram API credentials.

   - Add a **Gmail** node named `Email Workflow Stats`:
     - Send To: Use environment variable `EMAIL_ADDRESS_JOE`.
     - Subject: `"n8n User Memories"`
     - Message: From `$json.text`.
     - Connect input from `Prepare Gmail Message`.
     - Configure Gmail OAuth2 credentials.

8. **Create Tool Workflow Nodes for AI Agent:**

   For each tool below, add a **LangChain Tool Workflow** node with:

   - Name matching the tool function.
   - Field `route` set to the corresponding string.
   - Workflow ID set to this workflow's ID.
   - Description explaining the tool's purpose.

   Tools:

   - `🧠Save Memories` with route `"save_memory"`.
   - `🔎Retrieve Memories` with route `"retrieve_memory"`.
   - `📤Send Memories to Telegram` with route `"send_memories_to_telegram"`.
   - `📫Send Memories to Gmail` with route `"send_memories_to_gmail"`.

   Connect all these tool nodes as AI tools input to `🧠 AI Agent w/Long Term Memory`.

9. **Connect Workflow:**

   - Connect `When Executed by Another Workflow` to `Memory Tool Router`.
   - Connect outputs of `Memory Tool Router` to respective Google Docs nodes.
   - Connect Google Docs retrieval nodes to message preparation nodes.
   - Connect message preparation nodes to notification nodes.
   - Connect `Ⓜ️ When chat message received` to `🤖OpenAI Chat Model`, `🤯Window Buffer Memory`, and then to `🧠 AI Agent w/Long Term Memory`.
   - Connect AI agent outputs to tool workflow nodes.

10. **Set Environment Variables:**

    - `TELEGRAM_CHAT_ID` for Telegram notifications.
    - `EMAIL_ADDRESS_JOE` for Gmail notifications.

11. **Test and Deploy:**

    - Verify API credentials for OpenAI, Google Docs, Gmail, and Telegram.
    - Test saving and retrieving memories.
    - Test sending notifications.
    - Adjust system message and routing rules as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow empowers AI agents with long-term memory and dynamic tool routing, enhancing context-aware responses and multi-tool task management.                                                                                | Workflow description and purpose.                                                                            |
| Setup requires API credentials for OpenAI, Google Docs, Gmail, and Telegram.                                                                                                                                                        | Setup instructions section.                                                                                   |
| Customize the AI agent’s system message to tailor behavior and routing rules to fit specific use cases.                                                                                                                            | Customization instructions.                                                                                   |
| The workflow uses Google Docs as a persistent memory store, enabling recall of past interactions and preferences.                                                                                                                  | Long-term memory management.                                                                                   |
| Multi-channel notifications improve communication by sending memory summaries via Telegram and Gmail.                                                                                                                             | Notification dispatch block.                                                                                   |
| For more information on LangChain nodes and AI integrations in n8n, visit the official n8n documentation and LangChain resources.                                                                                                | External resources (not linked here).                                                                          |
| The workflow includes sticky notes within n8n for visual guidance on logical blocks and tool functions.                                                                                                                           | Sticky notes content preserved in node comments.                                                              |
| Environment variables `TELEGRAM_CHAT_ID` and `EMAIL_ADDRESS_JOE` must be set in n8n for notification nodes to function correctly.                                                                                                  | Environment variable requirements.                                                                             |

---

This document provides a detailed, structured reference to understand, reproduce, and modify the workflow efficiently, anticipating potential errors and integration points.