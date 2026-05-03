Create an autonomous task-handling AI agent with OpenAI and Slack

https://n8nworkflows.xyz/workflows/create-an-autonomous-task-handling-ai-agent-with-openai-and-slack-13822


# Create an autonomous task-handling AI agent with OpenAI and Slack

This technical analysis describes the "n8n Antigravity AI Automation: Create Autonomous Task-Handling AI Agent" workflow. This system implements a ReAct-style (Reason + Act) autonomous agent capable of iterative planning, tool execution, and self-reflection to complete complex tasks.

---

### 1. Workflow Overview

The workflow establishes a closed-loop system where an LLM (Large Language Model) acts as the brain, processing tasks received via Webhook. It breaks down tasks into steps, executes specific tools (HTTP requests or code), observes the results, and reflects on whether to continue or finish.

**Logical Blocks:**
*   **1.1 Input Reception:** Captures the task via Webhook and initializes the agent's memory and iteration counter.
*   **1.2 Initial Planning:** The LLM analyzes the task and generates the first action or a final answer.
*   **1.3 Agent Loop:** A recursive structure that parses LLM instructions, executes tools, merges observations, and updates the execution context.
*   **1.4 Output & Notification:** Formats the final resolution and sends a notification via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Receives the user request and prepares the data structure for the iterative process.
*   **Nodes Involved:** `Webhook: Start Agent`, `Initialize Task & Memory`.
*   **Node Details:**
    *   **Webhook: Start Agent:** Listens for a POST request at `/agent-run`. It expects a JSON body containing a `task` string.
    *   **Initialize Task & Memory (Set):** Defines the starting state. It sets `iteration` to 0, captures the `task` (with a default fallback), and initializes a `context` string to track history.

#### 2.2 Initial Planning
*   **Overview:** The first interaction with OpenAI to determine if the task is simple enough for an immediate answer or requires tool usage.
*   **Nodes Involved:** `LLM: Initial Plan`, `Already Finished?`.
*   **Node Details:**
    *   **LLM: Initial Plan (OpenAI):** Uses `gpt-4o-mini`. The system prompt enforces a strict JSON output schema (`thought`, `action`, `action_input`, `final_answer`).
    *   **Already Finished? (If):** Evaluates the JSON response. If the `action` is "finish", it bypasses the loop; otherwise, it triggers the iteration cycle.

#### 2.3 Agent Loop
*   **Overview:** The core "engine" where the agent performs actions and learns from results.
*   **Nodes Involved:** `Loop Control`, `Parse LLM Output`, `Loop → Finish?`, `Tool → HTTP Request`, `Other Tools (expand later)`, `Merge Observations`, `LLM: Reflect → Next`, `Update Memory`.
*   **Node Details:**
    *   **Loop Control (SplitInBatches):** Manages the iterations. While used here with a batch size of 1, its role is to facilitate the re-entry point for the loop.
    *   **Parse LLM Output (Code):** A safety layer that uses JavaScript to parse the LLM's string response into JSON and increments the iteration count.
    *   **Tool → HTTP Request:** Executed if the LLM requests a URL fetch. It uses dynamic expressions to set the URL from the LLM's `action_input`.
    *   **Other Tools (Code):** A placeholder for expanded capabilities (Web search, code execution). It generates an "observation" based on the simulated tool use.
    *   **LLM: Reflect → Next (OpenAI):** The "Reflect" stage. It takes the previous thought, the tool observation, and the task to decide the next step.
    *   **Update Memory (Set):** Appends the latest "Iteration | Thought | Action | Observation" to the context string, ensuring the LLM remembers previous attempts.

#### 2.4 Output & Notification
*   **Overview:** Finalizes the process once the LLM signals completion.
*   **Nodes Involved:** `Prepare Final Result`, `Wait For Result`, `Notify Slack`.
*   **Node Details:**
    *   **Prepare Final Result (Set):** Extracts the `final_answer` from the LLM or provides a fallback message.
    *   **Wait For Result (Wait):** A brief pause (default configuration) before final delivery.
    *   **Notify Slack:** Sends the task summary and the generated result to a specific Slack channel (`agent-results`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook: Start Agent | Webhook | Entry Point | (None) | Initialize Task & Memory | ## 1. Input |
| Initialize Task & Memory | Set | State Initialization | Webhook: Start Agent | LLM: Initial Plan | ## 1. Input |
| LLM: Initial Plan | OpenAI | Initial Reasoning | Initialize Task & Memory | Already Finished? | ## 2. Planning |
| Already Finished? | If | Logic Gate (Finish/Loop) | LLM: Initial Plan | Prepare Final Result, Loop Control | ## 2. Planning |
| Loop Control | SplitInBatches | Loop Management | Already Finished?, Update Memory | Parse LLM Output | ## 3. Agent Loop |
| Parse LLM Output | Code | Data Normalization | Loop Control | Loop → Finish? | ## 3. Agent Loop |
| Loop → Finish? | If | Logic Gate (Action/Finish) | Parse LLM Output | Prepare Final Result, Tool → HTTP Request, Other Tools | ## 3. Agent Loop |
| Tool → HTTP Request | HTTP Request | External Tool | Loop → Finish? | Merge Observations | ## 3. Agent Loop |
| Other Tools (expand later) | Code | Placeholder Tools | Loop → Finish? | Merge Observations | ## 3. Agent Loop |
| Merge Observations | Merge | Data Aggregation | Tool → HTTP Request, Other Tools | LLM: Reflect → Next | ## 3. Agent Loop |
| LLM: Reflect → Next | OpenAI | Iterative Reasoning | Merge Observations | Update Memory | ## 3. Agent Loop |
| Update Memory | Set | History Retention | LLM: Reflect → Next | Loop Control | ## 3. Agent Loop |
| Prepare Final Result | Set | Formatting | Already Finished?, Loop → Finish? | Wait For Result | ## 4. Output |
| Wait For Result | Wait | Delay | Prepare Final Result | Notify Slack | ## 4. Output |
| Notify Slack | Slack | Notification | Wait For Result | (None) | ## 4. Output |

---

### 4. Reproducing the Workflow from Scratch

1.  **Input Setup:** 
    *   Create a **Webhook** node (POST) to receive the `task`.
    *   Add a **Set** node to initialize variables: `iteration` (number: 0), `task` (string), and `context` (string).
2.  **Initial Logic:**
    *   Add an **OpenAI** node. Set the model to `gpt-4o-mini`. In the System Prompt, instruct it to output **only** JSON with keys: `thought`, `action`, `action_input`, and `final_answer`.
    *   Add an **If** node to check if `{{ $json.choices[0].message.content }}` contains `action === 'finish'`.
3.  **Building the Loop:**
    *   Add a **SplitInBatches** node (size 1) as the loop anchor.
    *   Add a **Code** node to parse the LLM response: `JSON.parse($json.choices[0].message.content)` and increment the iteration.
    *   Add an **If** node to route to "Tools" or "Finish".
4.  **Tool Integration:**
    *   Add an **HTTP Request** node. Set the URL using an expression from the parsed LLM `action_input`.
    *   Add a **Merge** node (Mode: Combine) to collect tool outputs.
    *   Add a second **OpenAI** node ("Reflect"). Pass the original task, current iteration, and current tool observation. Use the same JSON system prompt.
    *   Add a **Set** node to update the `context` variable by appending the new iteration details. Connect this back to the **SplitInBatches** node.
5.  **Output Logic:**
    *   Add a **Set** node ("Prepare Final Result") to capture the `final_answer`.
    *   Add a **Slack** node. Configure credentials and set the message to include the task and the result.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Model Choice:** The workflow uses `gpt-4o-mini` for speed/cost, but can be swapped for `gpt-4o` or `Claude 3.5 Sonnet` for better reasoning. | LLM Nodes |
| **Max Iterations:** To prevent infinite loops and cost overruns, it is recommended to add a hard limit check in the "Update Memory" or "Loop Control" node. | Safety / Guardrails |
| **Tool Expansion:** The "Other Tools" node is designed to be replaced by a **Switch** node leading to various integrations (Google Search, Databases, etc.). | Scalability |
| **Slack Channel:** Ensure the bot has permission to post in the `agent-results` channel. | Integration |