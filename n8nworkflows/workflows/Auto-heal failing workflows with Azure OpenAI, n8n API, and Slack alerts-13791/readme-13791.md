Auto-heal failing workflows with Azure OpenAI, n8n API, and Slack alerts

https://n8nworkflows.xyz/workflows/auto-heal-failing-workflows-with-azure-openai--n8n-api--and-slack-alerts-13791


# Auto-heal failing workflows with Azure OpenAI, n8n API, and Slack alerts

### 1. Workflow Overview

The **AI Self-Healing Engine & Auto-Patching System** is an advanced operational framework designed to minimize downtime in n8n environments. It acts as a global error handler that intercepts failures from other workflows, analyzes the root cause using Artificial Intelligence, and autonomously attempts to resolve the issue.

The workflow logic is divided into four functional blocks:
1.  **Error Interception & Context Retrieval:** Captures error metadata and fetches the complete source JSON of the failing workflow via the n8n REST API.
2.  **AI Diagnosis:** Uses an Azure OpenAI-powered agent to distinguish between transient failures (network timeouts) and structural failures (wrong parameters or logic errors).
3.  **Autonomous Resolution (Self-Healing):** 
    *   **Retry Logic:** If the error is transient, the system waits for a "cool down" period and triggers a re-execution.
    *   **Auto-Patching:** If the error is a logic/parameter issue, the system generates a corrected version of the workflow JSON and updates the production workflow live.
4.  **Notification & Reporting:** Dispatches detailed alerts to Slack regarding successful repairs or requests for manual intervention.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Interception & Context Retrieval
*   **Overview:** This block triggers upon any workflow failure and gathers the necessary technical context to allow the AI to "read" the code of the failing workflow.
*   **Nodes Involved:** `On Workflow Error`, `Filter: Ignore Self`, `Get Workflow JSON`.
*   **Node Details:**
    *   **On Workflow Error:** The entry point. It captures the execution ID, workflow ID, and the specific error message.
    *   **Filter: Ignore Self:** Ensures the healing engine doesn't attempt to heal itself if it fails, preventing infinite recursion loops. (Condition: `workflow.id` != current workflow ID).
    *   **Get Workflow JSON:** An HTTP Request node (GET) that calls the n8n API (`/workflows/{id}`). It requires an `X-N8N-API-KEY` header. Failure to provide a valid API key or incorrect URL will cause this block to fail.

#### 2.2 AI Diagnosis
*   **Overview:** The "brain" of the workflow. It processes the error message and the workflow structure to decide the best course of action.
*   **Nodes Involved:** `Diagnose Error`, `Azure OpenAI Chat Model1`, `Decision Schema`.
*   **Node Details:**
    *   **Diagnose Error (AI Agent):** Takes the failed node name, error message, and full Workflow JSON as input. It is prompted to act as a "Senior Engineer."
    *   **Azure OpenAI Chat Model1:** The LLM provider (using GPT-4o). Requires Azure credentials and deployment name.
    *   **Decision Schema (Output Parser):** Constraints the AI to return a specific JSON object containing a `state` ("RETRY" or "FIX"), a `diagnosis` string, and a `patch` object (parameter name and new value).

#### 2.3 Autonomous Resolution (Self-Healing)
*   **Overview:** Executes the technical fix based on the AI's verdict.
*   **Nodes Involved:** `Determine Action`, `Cool Down`, `Retry Execution`, `Generate Patch JSON`, `Update Workflow`.
*   **Node Details:**
    *   **Determine Action:** A Switch node that routes the flow based on the AI's `state` output.
    *   **Cool Down (Wait):** Pauses for a specified interval (e.g., 1 minute) to allow external services to recover.
    *   **Retry Execution:** Uses the n8n API (POST) to trigger a retry of the specific failed execution ID.
    *   **Generate Patch JSON (Code Node):** A JavaScript block that clones the original workflow JSON, finds the failing node, updates the identified faulty parameter with the AI's suggested value, and appends a "🛠 AI AUTO-FIX" note to the node for transparency.
    *   **Update Workflow:** Uses the n8n API (PUT) to overwrite the existing workflow with the newly patched JSON.

#### 2.4 Notification & Reporting
*   **Overview:** Informs the operations team of the outcome via Slack.
*   **Nodes Involved:** `Notify Success`, `Notify Manual Fix`.
*   **Node Details:**
    *   **Notify Success:** Sent when a "FIX" (patch) is successfully applied. Includes the AI's diagnosis.
    *   **Notify Manual Fix:** Sent if the AI suggests a manual review or if the "RETRY" logic is exhausted. Includes a direct link to the failed execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On Workflow Error | n8n-nodes-base.errorTrigger | Trigger | None | Filter: Ignore Self | The error trigger responds with failure in any workflow, then AI analyzes the failure context... |
| Filter: Ignore Self | n8n-nodes-base.filter | Safety Gate | On Workflow Error | Get Workflow JSON | The error trigger responds with failure in any workflow... |
| Get Workflow JSON | n8n-nodes-base.httpRequest | Data Acquisition | Filter: Ignore Self | Diagnose Error | The error trigger responds with failure in any workflow... |
| Diagnose Error | @n8n/n8n-nodes-langchain.agent | Logic/Reasoning | Get Workflow JSON | Determine Action | The error trigger responds with failure in any workflow... |
| Decision Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Data Formatting | None | Diagnose Error | N/A |
| Azure OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | AI Model | None | Diagnose Error | N/A |
| Determine Action | n8n-nodes-base.switch | Routing | Diagnose Error | Cool Down, Generate Patch JSON, Notify Manual Fix | The Switch node directs traffic based on the AI's verdict. |
| Cool Down | n8n-nodes-base.wait | Delay | Determine Action | Retry Execution | The Switch node directs traffic based on the AI's verdict. If a retry is needed... |
| Retry Execution | n8n-nodes-base.httpRequest | Action (Retry) | Cool Down | None | The Switch node directs traffic based on the AI's verdict. |
| Generate Patch JSON | n8n-nodes-base.code | Transformation | Determine Action | Update Workflow | The JavaScript node injects the AI's corrected value into the workflow JSON. |
| Update Workflow | n8n-nodes-base.httpRequest | Action (Update) | Generate Patch JSON | Notify Success | The JavaScript node injects the AI's corrected value into the workflow JSON. |
| Notify Success | n8n-nodes-base.slack | Notification | Update Workflow | None | The JavaScript node injects the AI's corrected value into the workflow JSON. |
| Notify Manual Fix | n8n-nodes-base.slack | Notification | Determine Action | None | If the AI cannot determine a fix or the error is marked as "MANUAL"... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Generate an n8n API Key from your User Settings.
    *   Ensure you have an Azure OpenAI Deployment (Model: GPT-4o).
    *   Prepare a Slack App with `chat:write` permissions.

2.  **Trigger & Filter Setup:**
    *   Add an **Error Trigger** node.
    *   Connect it to a **Filter** node. Set the condition to check that the incoming `workflow.id` (from the error) does not equal the current workflow ID to prevent loops.

3.  **Metadata Fetching:**
    *   Add an **HTTP Request** node.
    *   Method: `GET`. URL: `https://YOUR_N8N_DOMAIN/api/v1/workflows/{{ $json.workflow.id }}`.
    *   Header: `X-N8N-API-KEY` with your API key.

4.  **AI Integration:**
    *   Add an **AI Agent** node.
    *   Attach an **Azure OpenAI Chat Model** node.
    *   Attach a **Structured Output Parser** node. Define the schema with properties: `state` (enum: RETRY, FIX), `diagnosis` (string), and `patch` (object with `parameterName` and `newValue`).
    *   In the Agent prompt, pass the error message and the stringified JSON from the HTTP Request node.

5.  **Execution Routing:**
    *   Add a **Switch** node. Create three routing rules based on the AI's `state` output.

6.  **The "Retry" Branch:**
    *   Add a **Wait** node (e.g., 1-5 minutes).
    *   Add an **HTTP Request** node. Method: `POST`. URL: `.../api/v1/executions/{{ executionId }}/retry`.

7.  **The "Fix" Branch:**
    *   Add a **Code** node. Use JavaScript to parse the workflow JSON, locate the specific node mentioned in the AI output, and overwrite the parameter with the new value.
    *   Add an **HTTP Request** node. Method: `PUT`. URL: `.../api/v1/workflows/{{ workflowId }}`. Pass the updated JSON in the body.
    *   Add a **Slack** node to notify of the successful patch.

8.  **The "Manual" Branch:**
    *   Add a **Slack** node configured with a message containing the AI's diagnosis and a link to the execution for human review.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow acts as a global error handler. It must be selected as the "Error Workflow" in the settings of other workflows. | Workflow Settings > Error Workflow |
| Requires n8n API v1 enabled on your instance. | [n8n API Documentation](https://docs.n8n.io/api/v1/) |
| The "Generate Patch JSON" node adds a visual note to the modified node in the UI for auditing. | Logic implemented in the Code Node |
| Self-recursion check is critical: without the Filter node, a failure in this workflow would trigger itself infinitely. | Safety Configuration |