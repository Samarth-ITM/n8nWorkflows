Log workflow errors to Slack and Google Sheets

https://n8nworkflows.xyz/workflows/log-workflow-errors-to-slack-and-google-sheets-13789


# Log workflow errors to Slack and Google Sheets

# Reference Document: Log Workflow Errors to Slack and Google Sheets

### 1. Workflow Overview
The purpose of this workflow is to provide a centralized, automated error-handling mechanism for an n8n instance. It functions as an "Error Handler" workflow that triggers whenever another workflow in the same environment encounters an uncaught error.

The workflow captures execution metadata—such as the failing node's name, the error message, and a direct link to the execution—and performs two primary actions: it sends an immediate alert to a Slack channel and logs the event into a Google Sheet for long-term auditing and analysis.

**Logical Blocks:**
*   **1.1 Error Identification:** Captures the error context from the triggering workflow.
*   **1.2 Data Formatting:** Prepares the message strings and extracts relevant variables.
*   **1.3 Error Logging:** Disseminates the information to Slack and Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Identify Error
**Overview:** This block serves as the entry point for the workflow. It is passive until an error occurs in another workflow that points to this one as its "Error Workflow."

*   **Node Involved:**
    *   **Error Trigger**
*   **Node Details:**
    *   **Type:** Error Trigger (n8n-nodes-base.errorTrigger)
    *   **Configuration:** No specific parameters required. It automatically receives the `execution` and `workflow` objects from the failed process.
    *   **Input/Output:** No preceding nodes. Outputs a JSON object containing `execution` details (URL, error message, last node executed) and `workflow` details (ID, name).
    *   **Edge Cases:** If the failing workflow does not have its "Error Workflow" setting pointed to this workflow, this trigger will not fire.

#### 2.2 Message Formatting
**Overview:** This block transforms the raw error data into a human-readable format and prepares specific variables for the downstream logging nodes.

*   **Node Involved:**
    *   **Set message**
*   **Node Details:**
    *   **Type:** Set (n8n-nodes-base.set)
    *   **Configuration:** "Keep Only Set" is enabled to clean the data stream.
    *   **Key Expressions:**
        *   `message`: `:warning: [prod] workflow {{$json["workflow"]["name"]}} failed to run! <{{ $json.execution.url }}|execution> \n\nerror message from node: {{ $json.execution.lastNodeExecuted }} \n {{ $json.execution.error.message }}`
    *   **Input/Output:** Receives data from the Error Trigger; outputs a single `message` string and preserved execution metadata to Slack and Google Sheets.

#### 2.3 Log Error
**Overview:** This block performs the final integration actions, sending the formatted alert to Slack and appending the data to a spreadsheet.

*   **Nodes Involved:**
    *   **Slack**
    *   **Append row in sheet**
*   **Node Details (Slack):**
    *   **Type:** Slack (n8n-nodes-base.slack)
    *   **Configuration:** Send a message to a specific channel (ID/Name: `#alerts-n8n-workflows`).
    *   **Key Expressions:** `Text: {{ $json.message }}`.
    *   **Failure Types:** Authentication expired, Bot permissions missing in channel, or Slack API rate limits.
*   **Node Details (Append row in sheet):**
    *   **Type:** Google Sheets (n8n-nodes-base.googleSheets)
    *   **Configuration:** "Append" operation. Mapping is set to "Define below."
    *   **Mapped Columns:**
        *   `workflow`: `{{ $json.workflow.name }}`
        *   `url`: `{{ $json.execution.url }}`
        *   `node`: `{{ $json.execution.lastNodeExecuted }}`
        *   `error message`: `{{ $json.execution.error.message }}`
    *   **Failure Types:** OAuth2 token issues, Spreadsheet ID deleted, or column header mismatch.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Error Trigger** | Error Trigger | Entry Point | None | Set message | identify error |
| **Set message** | Set | Data Preparation | Error Trigger | Slack, Append row in sheet | identify error |
| **Slack** | Slack | Notification | Set message | None | log error |
| **Append row in sheet** | Google Sheets | Data Archiving | Set message | None | log error |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with headers: `workflow`, `url`, `node`, and `error message`.
    *   Create a Slack channel named `#alerts-n8n-workflows` and invite your n8n bot to it.

2.  **Step 1: The Trigger**
    *   Add an **Error Trigger** node. This requires no configuration.

3.  **Step 2: Data Formatting**
    *   Connect a **Set** node to the Error Trigger.
    *   Name it "Set message".
    *   Set "Mode" to "Define Below" and toggle "Keep Only Set" to **On**.
    *   Add a string value named `message`.
    *   Use an expression to build the Slack text, combining `$json.workflow.name`, `$json.execution.url`, `$json.execution.lastNodeExecuted`, and `$json.execution.error.message`.

4.  **Step 3: Slack Integration**
    *   Add a **Slack** node and connect it to the "Set message" node.
    *   Set the Resource to "Message" and Action to "Post".
    *   Select your Slack Credentials.
    *   Set the Channel to `#alerts-n8n-workflows`.
    *   In the Text field, use the expression `{{ $json.message }}`.

5.  **Step 4: Google Sheets Integration**
    *   Add a **Google Sheets** node and connect it to the "Set message" node (in parallel to Slack).
    *   Set the Operation to "Append".
    *   Select your Google Sheets Credentials.
    *   Select the Document and Sheet created in the preparation step.
    *   Set "Mapping Mode" to "Define Below".
    *   Map the headers (workflow, url, node, error message) to their corresponding expressions from the `$json.execution` and `$json.workflow` objects.

6.  **Final Step:**
    *   Save and **Activate** the workflow.
    *   Go to any other workflow settings and select this workflow as the "Error Workflow."

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Workflow Error Logger - How it works / Setup instructions | Contained within the main workflow Sticky Note |
| Identify Error | Structural label for the trigger phase |
| Log Error | Structural label for the action phase |