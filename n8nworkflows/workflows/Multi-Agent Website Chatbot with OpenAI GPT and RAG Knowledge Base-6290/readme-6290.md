Multi-Agent Website Chatbot with OpenAI GPT and RAG Knowledge Base

https://n8nworkflows.xyz/workflows/multi-agent-website-chatbot-with-openai-gpt-and-rag-knowledge-base-6290


# Multi-Agent Website Chatbot with OpenAI GPT and RAG Knowledge Base

### 1. Workflow Overview

This workflow implements a **Multi-Agent Website Chatbot** designed to handle customer inquiries on a website by intelligently routing requests to specialized AI sub-agents. It is suited for businesses wanting a scalable, modular chatbot that provides knowledgeable answers, schedules meetings, and escalates complex issues to human agents. The architecture uses a central orchestrator agent that leverages OpenAI GPT and a lightweight memory buffer to preserve conversational context, delegating tasks to dedicated sub-agents to optimize performance and maintainability.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming chat messages from the website interface.
- **1.2 Manager Agent (Ultimate Website Chatbot Agent):** Central language model agent that interprets user inputs and routes requests.
- **1.3 AI Model and Memory:** OpenAI GPT model for language understanding and a memory buffer to maintain recent conversation context.
- **1.4 Sub-Agent Routing:** Calls specialized workflows:
  - `calendarAgent` for calendar and booking tasks.
  - `RAGagent` for knowledge base FAQ retrieval.
  - `ticketAgent` for creating support tickets and human escalation.
- **1.5 Documentation and Setup Notes:** A sticky note node providing detailed architecture explanation and configuration guidelines.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for chat messages sent from the website widget via a webhook and triggers the workflow.
- **Nodes Involved:** `When chat message received`
- **Node Details:**

  - **Type & Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — webhook trigger for chat messages.
  - **Configuration:** Public webhook mode enabled, no additional options.
  - **Expressions/Variables:** Captures incoming JSON with user chat input, sessionId, and metadata.
  - **Connections:** Outputs to the `Ultimate Website Chatbot Agent`.
  - **Edge Cases:** Potential failures include webhook connectivity issues, malformed incoming data, or missing sessionId.
  - **Version:** 1.1
  - **Sub-workflow:** N/A

#### 1.2 Manager Agent (Ultimate Website Chatbot Agent)

- **Overview:** Acts as the orchestrator agent, processing incoming messages, deciding which sub-agent to call, enforcing business logic, and managing the overall conversation flow.
- **Nodes Involved:** `Ultimate Website Chatbot Agent`
- **Node Details:**

  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` — a multi-tool AI agent node.
  - **Configuration:** 
    - System message defines persona ("Dan Bot") and strict routing rules.
    - Lists available tools (`RAGagent`, `calendarAgent`, `ticketAgent`) with specific instructions on when to call each.
    - Enforces no improvisation; only tools answer queries.
    - Implements booking logic including timezone handling (Chicago) and slot availability.
    - Defines escalation criteria to human agents.
    - Sets conversational tone and default call to action (booking consultations).
  - **Expressions/Variables:** Utilizes current date/time in Chicago timezone; accesses sessionId from trigger node.
  - **Connections:** Inputs from `When chat message received`; outputs tool calls to sub-agents, memory updates to `Simple Memory`, and language model calls to `OpenAI Chat Model`.
  - **Edge Cases:** Failures can stem from invalid tool responses, API timeouts, or incomplete user input for booking.
  - **Version:** 1.7
  - **Sub-workflow:** N/A (manager agent node internally coordinates sub-agent calls)

#### 1.3 AI Model and Memory

- **Overview:** Supplies the core language model and short-term conversational memory to the manager agent.
- **Nodes Involved:** `OpenAI Chat Model`, `Simple Memory`
- **Node Details:**

  - **OpenAI Chat Model:**
    - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI GPT chat completion node.
    - **Configuration:** Uses model `gpt-4o-mini` (a lightweight GPT-4 variant).
    - **Credentials:** OpenAI API key configured.
    - **Connections:** Called by `Ultimate Website Chatbot Agent` as language model.
    - **Edge Cases:** API quota exhaustion, network errors, invalid API key.
    - **Version:** 1.2

  - **Simple Memory:**
    - **Type & Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — maintains a sliding window of last 10 messages to preserve context.
    - **Configuration:** Context window length set to 10 messages.
    - **Connections:** Linked as memory source for `Ultimate Website Chatbot Agent`.
    - **Edge Cases:** Memory overflow or loss of context if conversation is very long.
    - **Version:** 1.3

#### 1.4 Sub-Agent Routing

- **Overview:** Contains three specialized sub-agent workflows invoked by the manager agent for distinct functional domains.
- **Nodes Involved:** `calendarAgent`, `RAGagent`, `ticketAgent`
- **Node Details:**

  - **calendarAgent:**
    - **Type & Role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — external workflow to handle calendar availability and booking.
    - **Configuration:** Calls workflow ID `LpmYLHWdvevdwt5e`.
    - **Description:** Handles any calendar action including checking availability and booking meetings.
    - **Connections:** Invoked by `Ultimate Website Chatbot Agent` tool interface.
    - **Edge Cases:** Calendar API failures, double booking attempts, timezone mismatches.
    - **Version:** 2

  - **RAGagent:**
    - **Type & Role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — external workflow for Retrieval-Augmented Generation (RAG) FAQ answering.
    - **Configuration:** Calls workflow ID `IkEdDr98G9p54XDT`.
    - **Description:** Answers FAQs about the company by querying a knowledge base; input includes question and sessionId.
    - **Connections:** Invoked by `Ultimate Website Chatbot Agent`.
    - **Edge Cases:** Missing knowledge base entries, retrieval failures, long latency.
    - **Version:** 2

  - **ticketAgent:**
    - **Type & Role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — external workflow to generate support tickets for human follow-up.
    - **Configuration:** Calls workflow ID `tMyGGwgRHFuqYKg3`.
    - **Description:** Creates support tickets using user name, email, and request snippet.
    - **Connections:** Invoked by `Ultimate Website Chatbot Agent`.
    - **Edge Cases:** Email delivery failures, incomplete user info.
    - **Version:** 2

#### 1.5 Documentation and Setup Notes

- **Overview:** Provides comprehensive setup instructions, architectural overview, and benefits summary within the workflow canvas.
- **Nodes Involved:** `Sticky Note`
- **Node Details:**

  - **Type & Role:** `n8n-nodes-base.stickyNote` — documentation node.
  - **Configuration:** Large text block describing the architecture, setup steps, and usage recommendations.
  - **Connections:** None (purely informational).
  - **Edge Cases:** None.
  - **Version:** 1

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                              | Input Node(s)               | Output Node(s)                                 | Sticky Note                                                                                                                                                                                                                         |
|-------------------------------|-----------------------------------------|----------------------------------------------|-----------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received     | @n8n/n8n-nodes-langchain.chatTrigger   | Webhook trigger for incoming chat messages   | None                        | Ultimate Website Chatbot Agent                  |                                                                                                                                                                                                                                   |
| Ultimate Website Chatbot Agent | @n8n/n8n-nodes-langchain.agent          | Central orchestrator agent routing to tools  | When chat message received   | RAGagent, calendarAgent, ticketAgent, Simple Memory, OpenAI Chat Model |                                                                                                                                                                                                                                   |
| OpenAI Chat Model              | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Language model for chatbot reasoning          | Ultimate Website Chatbot Agent | Ultimate Website Chatbot Agent                   |                                                                                                                                                                                                                                   |
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains recent conversation context         | Ultimate Website Chatbot Agent | Ultimate Website Chatbot Agent                   |                                                                                                                                                                                                                                   |
| RAGagent                      | @n8n/n8n-nodes-langchain.toolWorkflow  | FAQ answering sub-agent                        | Ultimate Website Chatbot Agent | Ultimate Website Chatbot Agent                   | Call this tool to get answers for FAQs regarding Kamexa. The input should always be the question you want answered along with the following sessionId - {{ $('When chat message received').item.json.sessionId }}                   |
| calendarAgent                 | @n8n/n8n-nodes-langchain.toolWorkflow  | Calendar booking sub-agent                      | Ultimate Website Chatbot Agent | Ultimate Website Chatbot Agent                   | Call this tool for any calendar action.                                                                                                                                                                                          |
| ticketAgent                   | @n8n/n8n-nodes-langchain.toolWorkflow  | Support ticket creation sub-agent              | Ultimate Website Chatbot Agent | Ultimate Website Chatbot Agent                   | Call this tool to create a support ticket for a human agent to followup on via email. The input should be the users name and email address and a snippet of the users request.                                                     |
| Sticky Note                   | n8n-nodes-base.stickyNote               | Documentation and setup instructions           | None                        | None                                           | # Website Chatbot Agent with Modular Sub-Agent Architecture... (full detailed instructions and overview as in the node)                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**

   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Name: `When chat message received`
   - Configure as a public webhook (`mode: webhook`, `public: true`)
   - No additional options needed.
   - This node will serve as the entry point for user messages from the website.

2. **Create the Manager Agent Node**

   - Type: `@n8n/n8n-nodes-langchain.agent`
   - Name: `Ultimate Website Chatbot Agent`
   - Configure the system message with the detailed persona and routing rules:
     - Define the bot as "Dan Bot," explain available tools and their roles.
     - Include strict instructions not to answer without calling tools.
     - Set booking rules (Chicago timezone, 30-min slots).
     - Define escalation criteria to human agents.
     - Use friendly but focused tone.
     - Include dynamic date and timezone variables.
   - This agent will receive input from the chat trigger.

3. **Create the OpenAI Chat Model Node**

   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Name: `OpenAI Chat Model`
   - Set model to `gpt-4o-mini` or preferred GPT-4 variant.
   - Provide OpenAI API credentials.
   - Connect as the language model for the manager agent node.

4. **Create the Simple Memory Node**

   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
   - Name: `Simple Memory`
   - Set `contextWindowLength` to 10 messages.
   - Connect as memory buffer for the manager agent.

5. **Create Sub-Agent Tool Workflow Nodes**

   For each sub-agent, add a `@n8n/n8n-nodes-langchain.toolWorkflow` node:

   - **calendarAgent**
     - Name: `calendarAgent`
     - Set workflow ID to your calendar management sub-workflow.
     - Description: For calendar availability and booking.
     - Inputs: None defined (handled by sub-workflow).
     - Connect as a tool to the manager agent.

   - **RAGagent**
     - Name: `RAGagent`
     - Set workflow ID to your RAG FAQ sub-workflow.
     - Description: For answering FAQs using a knowledge base.
     - Inputs: Expects question and sessionId.
     - Connect as a tool to the manager agent.

   - **ticketAgent**
     - Name: `ticketAgent`
     - Set workflow ID to your support ticket creation sub-workflow.
     - Description: For creating support tickets for human agents.
     - Inputs: User name, email, and request snippet.
     - Connect as a tool to the manager agent.

6. **Connect Nodes**

   - Connect `When chat message received` main output to `Ultimate Website Chatbot Agent`.
   - Connect `Ultimate Website Chatbot Agent` outputs:
     - `ai_languageModel` to `OpenAI Chat Model`
     - `ai_memory` to `Simple Memory`
     - `ai_tool` to each sub-agent node (`calendarAgent`, `RAGagent`, `ticketAgent`)

7. **Create Sticky Note Documentation Node**

   - Type: `n8n-nodes-base.stickyNote`
   - Name: `Sticky Note`
   - Paste detailed architecture overview, setup instructions, and benefits.
   - Position on canvas for reference; no connections needed.

8. **Set Workflow Settings**

   - Timezone: `America/Chicago`
   - Execution order: Use default or `v1`
   - Caller policy: `workflowsFromSameOwner` (to limit execution context)

9. **Deploy**

   - Embed the chatbot frontend on your website pointing to the webhook URL from `When chat message received`.
   - Monitor and test functionality end-to-end.
   - Adjust OpenAI parameters and sub-agent workflows as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Modular sub-agent architecture improves scalability and maintainability by delegating responsibilities to specialized workflows.                                                                                                            | Workflow overview and design principle.                                                                         |
| Always configure your OpenAI API key securely in the `OpenAI Chat Model` node.                                                                                                                                                                | OpenAI API key management.                                                                                       |
| The chatbot uses Chicago timezone for all time-based interactions. Confirm this in prompts for consistency.                                                                                                                                   | Timezone handling in `Ultimate Website Chatbot Agent`.                                                         |
| To embed the chatbot, use a custom HTML widget or script connected to the webhook URL of `When chat message received`.                                                                                                                       | Frontend integration instructions.                                                                               |
| For support ticketing, ensure your ticketAgent workflow is configured with valid email or ticketing system credentials (e.g., SMTP, SendGrid).                                                                                               | Support escalation setup.                                                                                        |
| The `RAGagent` sub-workflow should connect to your internal knowledge base or document store via vector search or API for effective FAQ answering.                                                                                          | Knowledge base integration.                                                                                      |
| Monitoring usage via the n8n Executions tab is recommended to refine prompts and handle edge cases.                                                                                                                                           | Operational best practices.                                                                                      |
| Sticky Note contains full architectural explanation and setup instructions inside the workflow canvas for easy reference by team members.                                                                                                   | Documentation node on the canvas.                                                                                |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.