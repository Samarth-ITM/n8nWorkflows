Send structured logs to BetterStack from any workflow using HTTP Request

https://n8nworkflows.xyz/workflows/send-structured-logs-to-betterstack-from-any-workflow-using-http-request-3400


# Send structured logs to BetterStack from any workflow using HTTP Request

### 1. Workflow Overview

This workflow enables sending structured log messages to BetterStack Logs from any n8n workflow via HTTP requests. It is designed primarily for automation builders, developers, and DevOps teams who want centralized, consistent logging across multiple workflows without duplicating logging logic.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Receives log data (`level` and `message`) either from external workflows via an Execute Workflow Trigger or manually for testing.
- **1.2 Log Forwarding:** Sends the structured log data securely to BetterStack Logs using an HTTP POST request with header authentication.
- **1.3 Demonstration and Documentation:** Provides a manual trigger and an example Execute Workflow node to demonstrate usage, accompanied by sticky notes for guidance and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects log entries from external workflows or manual triggers. It standardizes input fields (`level` and `message`) for downstream processing.

**Nodes Involved:**  
- Recieve log message (Execute Workflow Trigger)  
- Test workflow (Manual Trigger)  
- Send test log message (Execute Workflow)

**Node Details:**

- **Recieve log message**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for log data sent from other workflows using Execute Workflow node.  
  - Configuration: Accepts workflow inputs named `level` and `message`.  
  - Expressions: None directly, but expects these inputs to be passed in.  
  - Inputs: None (trigger node)  
  - Outputs: Connected to "Send Log to BetterStack" node.  
  - Edge cases: Missing or malformed inputs may cause empty or invalid log messages; no explicit validation here.  
  - Version: 1.1  

- **Test workflow**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing purposes.  
  - Configuration: No parameters; triggers the workflow on user action.  
  - Inputs: None  
  - Outputs: Connected to "Send test log message" node.  
  - Edge cases: None significant; manual trigger is user-controlled.  
  - Version: 1  

- **Send test log message**  
  - Type: Execute Workflow  
  - Role: Demonstrates how to call this workflow from another workflow by passing sample log data.  
  - Configuration:  
    - Calls the current workflow by ID (`{{$workflow.id}}`).  
    - Passes inputs: `level` = "error", `message` = "This is a test log message".  
    - Input schema explicitly defines `level` and `message` as strings.  
  - Inputs: From "Test workflow" manual trigger.  
  - Outputs: None (end node)  
  - Edge cases: If the workflow ID changes or is incorrect, execution may fail.  
  - Version: 1.2  

---

#### 2.2 Log Forwarding

**Overview:**  
This block sends the received log data to BetterStack Logs via an authenticated HTTP POST request, ensuring secure and structured delivery.

**Nodes Involved:**  
- Send Log to BetterStack (HTTP Request)

**Node Details:**

- **Send Log to BetterStack**  
  - Type: HTTP Request  
  - Role: Sends structured log data to BetterStack Logs ingestion endpoint.  
  - Configuration:  
    - Method: POST  
    - Body: JSON, constructed using expressions to insert `message` and `level` from incoming data:  
      ```json
      {
        "message": "{{ $json.message }}",
        "level": "{{ $json.level }}"
      }
      ```  
    - Authentication: HTTP Header Auth via a credential named "Header Auth BetterStack".  
    - URL: User must replace with their BetterStack Logs ingestion URL.  
  - Inputs: From "Recieve log message" node.  
  - Outputs: None (end node)  
  - Edge cases:  
    - Authentication failure if API key is invalid or missing.  
    - Network timeouts or endpoint unavailability.  
    - Malformed JSON if input data is missing or improperly formatted.  
  - Version: 4.2  

---

#### 2.3 Demonstration and Documentation

**Overview:**  
This block provides user guidance, setup instructions, and demonstration notes via sticky notes to facilitate understanding and customization.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Explains usage modes (standalone or shared workflow).  
  - Content:  
    ```
    ## Send log entries to BetterStack
    This workflow can be used in two ways:
    1. Save it as a separate workflow to
    use if from multiple worflows.
    2. Embed it into one workflow to just
    use it from one.
    ```  
  - Position: Near input nodes for visibility.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Demo explanation for the test call node.  
  - Content:  
    ```
    ## Demo
    This is just a demo of how to call the workflow.
    Keep it here, replace it with your own workflow or delete it.
    ```  

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Placeholder note labeled "Edit me" for user customization.  

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Author introduction, usage instructions, and setup guidance.  
  - Content:  
    ```
    ### 🧾 Log to BetterStack

    **👋 Hello! I'm Audun / xqus** 
    🔗 My work: [xqus.com](https://xqus.com)
    💸 n8n shop: [xqus.gumroad.com](https://xqus.gumroad.com)

    This workflow sends log messages to [BetterStack Logs](https://betterstack.com/logs) using a POST request.

    #### ✅ Usage:
    1. **From other workflows**  
       → Use the **Execute Workflow** node and pass in `level` and `message`.

    2. **As standalone**  
       → Manually trigger for testing, or embed it into a single workflow.

    #### 🔧 Setup:
    1. Set your **BetterStack Logs endpoint URL** in the HTTP Request node.  
    2. Add your **Header Auth** credentials: `Authorization: Bearer YOUR_TOKEN`
    ```  

---

### 3. Summary Table

| Node Name             | Node Type              | Functional Role                             | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                         |
|-----------------------|------------------------|---------------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------|
| Recieve log message    | Execute Workflow Trigger | Receives log inputs (`level`, `message`) from other workflows | None                   | Send Log to BetterStack  | from another workflow                                                                                               |
| Send Log to BetterStack| HTTP Request           | Sends structured log to BetterStack Logs via POST with header auth | Recieve log message    | None                     |                                                                                                                     |
| Test workflow         | Manual Trigger          | Manual start for testing the workflow       | None                   | Send test log message     |                                                                                                                     |
| Send test log message  | Execute Workflow        | Calls this workflow with sample log data    | Test workflow          | None                     | using workflow                                                                                                      |
| Sticky Note           | Sticky Note             | Usage modes explanation                      | None                   | None                     | ## Send log entries to BetterStack This workflow can be used in two ways: 1. Save it as a separate workflow to use if from multiple worflows. 2. Embed it into one workflow to just use it from one. |
| Sticky Note1          | Sticky Note             | Demo explanation for test call               | None                   | None                     | ## Demo This is just a demo of how to call the workflow. Keep it here, replace it with your own workflow or delete it. |
| Sticky Note2          | Sticky Note             | Placeholder for user customization           | None                   | None                     | ### Edit me                                                                                                         |
| Sticky Note3          | Sticky Note             | Author intro and detailed setup instructions | None                   | None                     | ### 🧾 Log to BetterStack **👋 Hello! I'm Audun / xqus** 🔗 My work: [xqus.com](https://xqus.com) 💸 n8n shop: [xqus.gumroad.com](https://xqus.gumroad.com) This workflow sends log messages to [BetterStack Logs](https://betterstack.com/logs) using a POST request. #### ✅ Usage: 1. **From other workflows** → Use the **Execute Workflow** node and pass in `level` and `message`. 2. **As standalone** → Manually trigger for testing, or embed it into a single workflow. #### 🔧 Setup: 1. Set your **BetterStack Logs endpoint URL** in the HTTP Request node. 2. Add your **Header Auth** credentials: `Authorization: Bearer YOUR_TOKEN` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Recieve log message" node:**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: Add two workflow inputs named `level` and `message` (both strings).  
   - Position: Left side of canvas.

2. **Create the "Send Log to BetterStack" node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: (Leave blank for now; user must replace with their BetterStack Logs ingestion URL)  
   - Authentication: Select "HTTP Header Auth" credential (create this credential with header `Authorization: Bearer YOUR_API_KEY`)  
   - Body Content: JSON  
   - Body Parameters: Use expression mode and enter:  
     ```json
     {
       "message": "{{ $json.message }}",
       "level": "{{ $json.level }}"
     }
     ```  
   - Connect output of "Recieve log message" node to input of this node.

3. **Create the "Test workflow" node:**  
   - Type: Manual Trigger  
   - No parameters needed.  
   - Position: Top-left area.

4. **Create the "Send test log message" node:**  
   - Type: Execute Workflow  
   - Workflow to execute: Set to the current workflow (use expression `{{$workflow.id}}`)  
   - Workflow Inputs: Define schema with two string fields: `level` and `message`.  
   - Pass values: `level` = "error", `message` = "This is a test log message".  
   - Connect output of "Test workflow" node to input of this node.

5. **Create Sticky Notes:**  
   - Add four sticky notes with the following contents and place them near relevant nodes for clarity:  
     - Usage modes explanation (near input nodes).  
     - Demo explanation (near test nodes).  
     - Placeholder "Edit me" note.  
     - Author introduction and detailed setup instructions.

6. **Create HTTP Header Auth Credential:**  
   - In n8n credentials, create a new HTTP Header Auth credential named "Header Auth BetterStack".  
   - Add header: `Authorization: Bearer YOUR_API_KEY` (replace `YOUR_API_KEY` with your actual BetterStack API key).

7. **Final Setup:**  
   - Replace the URL in "Send Log to BetterStack" node with your BetterStack Logs ingestion endpoint URL.  
   - Save the workflow.

8. **Usage:**  
   - To send logs from other workflows, use an Execute Workflow node targeting this workflow, passing `level` and `message`.  
   - For testing, trigger "Test workflow" manually to send a sample error log.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow sends log messages to [BetterStack Logs](https://betterstack.com/logs) using a POST request with secure header authentication. It is authored by Audun / xqus. For more of his work, visit [xqus.com](https://xqus.com) or the n8n shop at [xqus.gumroad.com](https://xqus.gumroad.com).                                                                 | Author and project info from Sticky Note3        |
| The workflow is designed to be reusable either as a standalone logging workflow or embedded within other workflows for centralized log management.                                                                                                                                                                                                                 | Usage note from Sticky Note                        |
| BetterStack Logs documentation and API details can be found at https://betterstack.com/logs to assist with endpoint setup and log format customization.                                                                                                                                                                                                           | External resource link                            |
| Ensure your BetterStack API key is kept secure and never exposed in public workflows or logs. Use n8n's credential management for safe storage.                                                                                                                                                                                                                     | Security best practice                            |

---

This document fully describes the workflow’s structure, logic, and setup, enabling both human users and AI agents to understand, reproduce, and modify the workflow effectively.