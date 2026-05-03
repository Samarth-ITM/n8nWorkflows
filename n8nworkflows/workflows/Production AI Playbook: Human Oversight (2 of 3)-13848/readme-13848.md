Production AI Playbook: Human Oversight (2 of 3)

https://n8nworkflows.xyz/workflows/production-ai-playbook--human-oversight--2-of-3--13848


# Production AI Playbook: Human Oversight (2 of 3)

# Production AI Playbook: Human Oversight (Reference Document)

### 1. Workflow Overview

This workflow implements a Sales Pipeline Assistant designed to manage deals and automate sensitive actions like sending contracts. Its primary purpose is to demonstrate **Human-in-the-loop (HITL)** patterns in AI systems. The workflow ensures that high-risk actions (e.g., sending legal documents) are intercepted and require manual approval via a chat interface before execution.

The logic is organized into three functional blocks:
1.  **Input & Interaction:** Captures user requests via a chat interface.
2.  **AI Intelligence:** An AI Agent powered by a Large Language Model (LLM) that interprets intent and decides which tools to call.
3.  **Human Oversight Gate:** A middle layer that intercepts tool calls, presenting them to a human for approval or rejection.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Interaction
*   **Overview:** This block serves as the entry point for the user to interact with the sales assistant.
*   **Nodes Involved:** `When chat message received`
*   **Node Details:**
    *   **When chat message received (Chat Trigger):** 
        *   **Type:** `@n8n/n8n-nodes-langchain.chatTrigger` (v1.4)
        *   **Configuration:** Configured in `responseNodes` mode, meaning it waits for the nodes connected to the agent to provide the final output. File uploads are disabled.
        *   **Input/Output:** Receives raw text from the user; outputs to the AI Agent.
        *   **Failure Modes:** Webhook URL mismatch or session timeout in the chat interface.

#### 2.2 AI Intelligence
*   **Overview:** The "brain" of the workflow. It maintains the system persona and determines if a tool (like "Send Contract") should be used based on the user's prompt.
*   **Nodes Involved:** `AI Agent`, `OpenRouter Chat Model`
*   **Node Details:**
    *   **AI Agent (Agent):**
        *   **Type:** `@n8n/n8n-nodes-langchain.agent` (v3.1)
        *   **Configuration:** Utilizes a System Message defining it as a "sales pipeline assistant." It is explicitly instructed to use the `Send Contract` tool when deal details (ID, email, name) are provided.
        *   **Key Expression:** System Message: *"You are a sales pipeline assistant... Use the Send Contract tool to send the contract."*
        *   **Connections:** Receives input from the Chat Trigger and Model; connects to the HITL tool gate.
    *   **OpenRouter Chat Model (Chat Model):**
        *   **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (v1)
        *   **Configuration:** Connects to OpenRouter to provide LLM capabilities. Default settings are used.
        *   **Credentials:** Requires `openRouterApi`.
        *   **Failure Modes:** API quota exhaustion, invalid API key, or model latency.

#### 2.3 Human Oversight Gate & Tool Execution
*   **Overview:** This block acts as a security and quality control layer. It prevents the AI from executing the "Send Contract" code until a human clicks "Approve."
*   **Nodes Involved:** `Review: Send Contract`, `Send Contract`
*   **Node Details:**
    *   **Review: Send Contract (Human-in-the-loop Tool):**
        *   **Type:** `@n8n/n8n-nodes-langchain.chatHitlTool` (v1.2)
        *   **Configuration:** Approval type set to `double` (requiring explicit confirmation). It intercepts the call intended for the "Send Contract" tool.
        *   **Input/Output:** Sits between the AI Agent and the Code Tool.
        *   **Failure Modes:** User ignores the approval request; session expires.
    *   **Send Contract (Code Tool):**
        *   **Type:** `@n8n/n8n-nodes-langchain.toolCode` (v1.3)
        *   **Logic:** A JavaScript snippet that parses the input query (JSON string) containing `dealId` and `prospectEmail`. It returns a confirmation string.
        *   **Key Logic:** `return Contract sent to ${data.prospectEmail} for deal ${data.dealId};`
        *   **Failure Modes:** Invalid JSON input from the LLM or missing fields in the user request.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When chat message received | Chat Trigger | Input Entry Point | - | AI Agent | Sales Pipeline Agent with Tool Call Approval Gates |
| AI Agent | Agent | Core Logic / Orchestrator | When chat message received, OpenRouter Model | Review: Send Contract | Sales Pipeline Agent with Tool Call Approval Gates |
| OpenRouter Chat Model | Chat Model | LLM Provider | - | AI Agent | Sales Pipeline Agent with Tool Call Approval Gates |
| Review: Send Contract | HITL Tool | Human Approval Gate | AI Agent | Send Contract | Sales Pipeline Agent with Tool Call Approval Gates |
| Send Contract | Code Tool | Action Execution | Review: Send Contract | AI Agent (via Gate) | Sales Pipeline Agent with Tool Call Approval Gates |

---

### 4. Reproducing the Workflow from Scratch

1.  **Setup Chat Interface:**
    *   Add the **When chat message received** node. 
    *   Set *Public Preview Response Mode* to "Using 'Respond to Webhook' node" (or leave as default for standard chat behavior).

2.  **Configure the AI Brain:**
    *   Add an **AI Agent** node.
    *   In the **System Message**, define the persona: "You are a sales pipeline assistant... use the Send Contract tool when deal details are provided."
    *   Connect the Chat Trigger to the AI Agent.

3.  **Connect the Language Model:**
    *   Add an **OpenRouter Chat Model** node (or OpenAI/Anthropic equivalent).
    *   Configure credentials and connect it to the AI Agent's "Language Model" input.

4.  **Create the Protected Tool:**
    *   Add a **Code Tool** node. Name it "Send Contract".
    *   Set the **Description** to: "Sends a contract to a prospect. Input should be a JSON string with dealId and prospectEmail fields."
    *   Insert JavaScript to process the input:
        ```javascript
        const data = JSON.parse(query);
        return `Contract sent to ${data.prospectEmail} for deal ${data.dealId}`;
        ```

5.  **Implement the Approval Gate:**
    *   Add a **Human-in-the-loop (HITL)** node (Chat HITL Tool). Name it "Review: Send Contract".
    *   Set **Approval Type** to "Double".
    *   **Crucial Connection:** Connect the **AI Agent** to the **Review: Send Contract** node, and then connect the **Review: Send Contract** node to the **Send Contract** tool.

6.  **Testing:**
    *   Open the n8n Chat UI.
    *   Prompt: "Send the Enterprise contract to jane@doe.com for deal 9988".
    *   Observe the "Approve/Decline" buttons appearing in the chat before the code executes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Project Context** | This template is part of the Production AI Playbook series focusing on reliable AI systems. |
| **Strategy Guide** | Detailed strategies for Human Oversight and HITL patterns in n8n. [Link to blog](https://go.n8n.io/PAP-HO-Blog) |
| **Implementation Tip** | Only gate high-risk tools (payments, legal sends) to maintain a balance between autonomy and security. |