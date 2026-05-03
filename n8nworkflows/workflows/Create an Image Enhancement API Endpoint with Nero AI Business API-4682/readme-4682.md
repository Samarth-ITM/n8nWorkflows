Create an Image Enhancement API Endpoint with Nero AI Business API

https://n8nworkflows.xyz/workflows/create-an-image-enhancement-api-endpoint-with-nero-ai-business-api-4682


# Create an Image Enhancement API Endpoint with Nero AI Business API

### 1. Workflow Overview

This workflow creates an API endpoint to perform image enhancement or processing tasks using the Nero AI Business API.  
It accepts incoming HTTP POST requests via a webhook, creates an AI task on Nero’s platform (e.g., face detection or other AI services), polls the task status until completion, and then responds back to the original request with the processed results.

Logical blocks:

- **1.1 Input Reception:** Receives incoming requests through a webhook and initiates the AI task creation.  
- **1.2 AI Task Creation:** Sends a task creation request to Nero AI API with specified parameters and an API key.  
- **1.3 Task Polling:** Waits and queries the task status repeatedly until it completes successfully.  
- **1.4 Response Handling:** Sends the final AI task result back as the webhook response.  
- **1.5 Workflow Guidance:** Sticky notes provide instructions on usage and configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for incoming HTTP POST requests and triggers the workflow to start the AI processing.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  

  - **Webhook**  
    - Type: Webhook (HTTP endpoint)  
    - Role: Entry point for external POST requests to trigger the workflow.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook ID path (`c9795945-7dbb-45cb-9082-d3629f15504a`)  
      - Response Mode: Waits for the "Respond to Webhook" node to send the response.  
    - Inputs: External HTTP client  
    - Outputs: Connected to "Create task" node  
    - Edge Cases: Invalid HTTP methods, malformed requests, or timeouts if downstream nodes fail.  
    - Notes: This node exposes the API endpoint.

---

#### 2.2 AI Task Creation

- **Overview:**  
  Creates an AI task on Nero AI platform by sending a POST request with the desired service parameters and API key.

- **Nodes Involved:**  
  - Create task

- **Node Details:**  

  - **Create task**  
    - Type: HTTP Request  
    - Role: Sends the AI task creation request to Nero API.  
    - Configuration:  
      - URL: `https://ai.nero.com/biz/api/task`  
      - Method: POST  
      - Headers: Includes `x-neroai-api-key` with the user’s API key (must be replaced with valid key)  
      - Body: JSON specifying AI service type (default is `"FaceDetection"`) and input images array with placeholder URL (`https://image.url`)  
      - Sends JSON body and headers.  
    - Inputs: From "Webhook" node  
    - Outputs: To "Wait" node  
    - Edge Cases:  
      - Authentication errors (invalid or missing API key)  
      - API rate limits or service errors  
      - Incorrect JSON body structure or missing parameters  
    - Notes: User must customize the task type and input images per use case.

---

#### 2.3 Task Polling

- **Overview:**  
  Implements a polling mechanism to check the status of the AI task until it completes.

- **Nodes Involved:**  
  - Wait  
  - Query task status  
  - If

- **Node Details:**  

  - **Wait**  
    - Type: Wait node  
    - Role: Delays execution by 2 seconds before querying task status again.  
    - Configuration: Wait time of 2 seconds  
    - Inputs: From "Create task" or "If" (on incomplete task)  
    - Outputs: To "Query task status"  
    - Edge Cases:  
      - Delays workflow unnecessarily if configured too high  
      - Potential timeout if Nero API delays response excessively

  - **Query task status**  
    - Type: HTTP Request  
    - Role: Queries Nero API for the status of the previously created task.  
    - Configuration:  
      - URL: `https://ai.nero.com/biz/api/task`  
      - Method: GET with query parameter `task_id` extracted from previous node output (`{{$json.data.task_id}}`)  
      - Header: Includes API key header `x-neroai-api-key`  
    - Inputs: From "Wait" node  
    - Outputs: To "If" node  
    - Edge Cases:  
      - Network or API errors  
      - Missing or expired task_id  
      - Authentication failures

  - **If**  
    - Type: Conditional Check  
    - Role: Checks if the task is completed successfully.  
    - Configuration:  
      - Condition 1: `$json.code` equals `0` (indicates success)  
      - Condition 2: `$json.data.status` equals `done` (task completed)  
      - Both conditions must be true to proceed to response.  
    - Inputs: From "Query task status"  
    - Outputs:  
      - True branch: To "Respond to Webhook" node (send result back)  
      - False branch: To "Wait" (continue polling)  
    - Edge Cases:  
      - Task failure or error states not handled explicitly  
      - Infinite looping if task never completes

---

#### 2.4 Response Handling

- **Overview:**  
  Sends the final AI task result as the HTTP response for the original webhook request.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Returns the final data from Nero API back to the client who called the webhook.  
    - Configuration: Default options; sends data from "If" node output.  
    - Inputs: From "If" node (true branch)  
    - Outputs: None (endpoint response)  
    - Edge Cases:  
      - If the workflow fails before this node, no response will be sent, causing client timeouts.

---

#### 2.5 Workflow Guidance (Sticky Notes)

- **Overview:**  
  Provides user instructions and workflow description within n8n canvas.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  

  - **Sticky Note**  
    - Content: Describes the workflow’s purpose as an AI task API endpoint using Webhook and Respond to Webhook nodes.  
    - Position: Upper left of canvas

  - **Sticky Note1**  
    - Content:  
      - Steps to obtain API key from https://ai.nero.com/business  
      - Instructions to fill API key in HTTP nodes  
      - Reference to Nero AI API documentation: https://ai.nero.com/ai-api/docs  
      - How to test the endpoint with Postman or other tools  
    - Position: Below the first sticky note

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                       |
|---------------------|----------------------|-------------------------------|-----------------------|------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook              | API entry point                | External HTTP client  | Create task             |                                                                                                                  |
| Create task         | HTTP Request         | Create AI task on Nero API     | Webhook               | Wait                    |                                                                                                                  |
| Wait                | Wait                 | Delay before polling status    | Create task, If (false branch) | Query task status        |                                                                                                                  |
| Query task status    | HTTP Request         | Check AI task status           | Wait                  | If                      |                                                                                                                  |
| If                  | If                   | Check if task completed        | Query task status      | Respond to Webhook (true), Wait (false) |                                                                                                                  |
| Respond to Webhook   | Respond to Webhook   | Return final response          | If (true branch)       | None                    |                                                                                                                  |
| Sticky Note         | Sticky Note          | Workflow purpose explanation   | None                  | None                    | "Create an AI task API endpoint\n\nIn this workflow we show how to create an AI task API endpoint with `Webhook` and `Respond to Webhook` nodes" |
| Sticky Note1        | Sticky Note          | Usage instructions             | None                  | None                    | "### How to use it\n1. Apply for an API key from https://ai.nero.com/business\n2. Fill your key into the `Create task` and `Query task status` nodes\n3. Select an AI service and modify `Create task` node parameters, the API doc: https://ai.nero.com/ai-api/docs\n4. Execute the workflow so that the webhook starts listening\n5. Make a test request by postman or other tools, the test URL from the `Webhook` node\n\nYou will receive the output in the webhook response." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: A unique identifier or string (e.g., `c9795945-7dbb-45cb-9082-d3629f15504a`)  
   - Response Mode: Set to “Response Node” (wait for Respond to Webhook node)  
   - Position on canvas: Left/top area

2. **Create HTTP Request Node “Create task”**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://ai.nero.com/biz/api/task`  
   - Authentication: None (use header)  
   - Headers: Add header `x-neroai-api-key` with your Nero AI API key  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "type": "FaceDetection",
       "body": {
         "images": [
           "https://image.url"
         ]
       }
     }
     ```  
   - Connect input from Webhook node.

3. **Create Wait Node**  
   - Type: Wait  
   - Wait Time: 2 seconds  
   - Connect input from “Create task” node.

4. **Create HTTP Request Node “Query task status”**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://ai.nero.com/biz/api/task`  
   - Authentication: None (use header)  
   - Headers: Add header `x-neroai-api-key` with your API key  
   - Query Parameters:  
     - `task_id` set to expression `{{$json.data.task_id}}` (retrieved from previous node output)  
   - Connect input from Wait node.

5. **Create If Node**  
   - Type: If  
   - Condition (AND):  
     - `$json.code` equals `0` (number)  
     - `$json.data.status` equals `done` (string)  
   - Connect input from “Query task status” node.  
   - True branch: Connect to “Respond to Webhook” node.  
   - False branch: Connect back to “Wait” node for polling.

6. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Default configuration (sends incoming data as response)  
   - Connect input from If node (true branch).

7. **(Optional) Add Sticky Notes**  
   - Add notes for workflow description and usage instructions, including:  
     - API key application link: https://ai.nero.com/business  
     - API documentation: https://ai.nero.com/ai-api/docs  
     - How to test webhook URL with Postman or similar tools.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                            | Context or Link                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------|
| Apply for an API key at https://ai.nero.com/business to use Nero AI Business API.                                                                                                                                       | Official API key application page |
| Nero AI API documentation detailing AI service types and parameters: https://ai.nero.com/ai-api/docs                                                                                                                   | API docs for modifying task types |
| The webhook response mode is set to wait for the “Respond to Webhook” node, ensuring synchronous API response.                                                                                                          | Webhook configuration best practice |
| Typical use case: image enhancement, face detection, or other AI vision tasks by changing the `type` and `body` parameters in the “Create task” HTTP node.                                                              | Use-case customization guide     |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.