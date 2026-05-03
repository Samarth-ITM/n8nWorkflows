Send n8n Error Reports to LINE

https://n8nworkflows.xyz/workflows/send-n8n-error-reports-to-line-3141


# Send n8n Error Reports to LINE

### 1. Workflow Overview

This workflow, titled **"n8n Error Report to Line"**, is designed to provide real-time error notifications from n8n workflows directly to a LINE messaging account. It targets users who want to monitor and respond quickly to workflow failures by leveraging the LINE Messaging API.

The workflow consists of two main logical blocks:

- **1.1 Error Detection Block:** Captures any error occurring in n8n workflows configured to use this error workflow.
- **1.2 Notification Dispatch Block:** Sends a formatted message containing error details to a specified LINE user via the LINE Push API.

This structure ensures seamless error monitoring by automatically triggering on errors and pushing notifications to the user’s LINE account.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection Block

- **Overview:**  
  This block listens for any error events triggered by n8n workflows. It acts as the entry point for the workflow, automatically activating when an error occurs in any workflow that has this error workflow set as its error handler.

- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**

  - **Node Name:** Error Trigger  
    - **Type:** Error Trigger (n8n built-in node)  
    - **Technical Role:** Listens for errors in other workflows and triggers this error-handling workflow.  
    - **Configuration:** Default settings; no additional parameters required.  
    - **Key Expressions/Variables:** Provides error context in JSON format, including workflow name and execution URL accessible via `$json.workflow.name` and `$json.execution.url`.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Outputs to the HTTP Request node.  
    - **Version Requirements:** Compatible with n8n versions supporting error workflows (generally v0.140.0+).  
    - **Potential Failures:** Misconfiguration if this workflow is not set as the error workflow; no output if no errors occur.  
    - **Sub-workflow:** None.

#### 2.2 Notification Dispatch Block

- **Overview:**  
  This block sends an HTTP POST request to the LINE Messaging API to push a notification message to the user’s LINE account. The message includes the workflow name and a URL to the failed execution for quick troubleshooting.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **Node Name:** HTTP Request  
    - **Type:** HTTP Request  
    - **Technical Role:** Sends a POST request to the LINE Push API endpoint to deliver a text message.  
    - **Configuration:**  
      - **URL:** `https://api.line.me/v2/bot/message/push`  
      - **Method:** POST  
      - **Authentication:** HTTP Header Authentication using a pre-configured credential named "Line @271dudsw MiniBear".  
      - **Request Body:** JSON formatted message containing:  
        ```json
        {
          "to": "<UID HERE>",
          "messages": [
            {
              "type": "text",
              "text": "🚨Your n8n flow is dead.🚨\n\nName: {{ $json.workflow.name }} \nURL: {{ $json.execution.url }}"
            }
          ]
        }
        ```  
      - **Expressions:** Uses mustache-style expressions to inject dynamic error data (`$json.workflow.name` and `$json.execution.url`).  
      - **Send Body:** Enabled, specifying JSON content.  
    - **Input Connections:** Receives data from the Error Trigger node.  
    - **Output Connections:** None (terminal node).  
    - **Version Requirements:** Requires n8n version supporting HTTP Request node v4.2 or later for JSON body and authentication features.  
    - **Potential Failures:**  
      - Authentication errors if LINE credentials are invalid or expired.  
      - HTTP errors if the LINE API endpoint is unreachable or rate-limited.  
      - Expression evaluation errors if error context data is missing.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name       | Node Type       | Functional Role               | Input Node(s)   | Output Node(s) | Sticky Note                                                                                                      |
|-----------------|-----------------|------------------------------|-----------------|----------------|------------------------------------------------------------------------------------------------------------------|
| Error Trigger   | Error Trigger   | Detects workflow errors       | None            | HTTP Request   | ## Error Trigger\n\nThis flow will get trigger when the error occur. You can set only one error flow for all your flows. |
| HTTP Request    | HTTP Request    | Sends LINE notification       | Error Trigger   | None           | ## Send Line Message\n\nTo send message to notify you via Line Account -- Please replace <UID HERE> with your own UID |
| Sticky Note     | Sticky Note     | Instructional note            | None            | None           | ## Error Handling\n\nYou can set this workflow as error workflow\n\nhttps://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow |
| Sticky Note1    | Sticky Note     | Instructional note            | None            | None           | ## Error Trigger\n\nThis flow will get trigger when the error occur. You can set only one error flow for all your flows. |
| Sticky Note2    | Sticky Note     | Instructional note            | None            | None           | ## Send Line Message\n\nTo send message to notify you via Line Account -- Please replace <UID HERE> with your own UID |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** and name it "n8n Error Report to Line".

2. **Add the Error Trigger node:**  
   - Node Type: **Error Trigger**  
   - Position: Place it on the left side of the canvas.  
   - Configuration: No parameters needed; default settings suffice.  
   - This node will automatically trigger when any error occurs in workflows that use this error workflow.

3. **Add the HTTP Request node:**  
   - Node Type: **HTTP Request**  
   - Position: To the right of the Error Trigger node.  
   - Connect the output of the Error Trigger node to the input of this HTTP Request node.

4. **Configure the HTTP Request node:**  
   - **HTTP Method:** POST  
   - **URL:** `https://api.line.me/v2/bot/message/push`  
   - **Authentication:**  
     - Select **HTTP Header Auth**.  
     - Create or select credentials for your LINE bot (see below).  
   - **Body Content:**  
     - Set **Body Content Type** to JSON.  
     - Use the following JSON body with expressions:  
       ```json
       {
         "to": "<UID HERE>",
         "messages": [
           {
             "type": "text",
             "text": "🚨Your n8n flow is dead.🚨\n\nName: {{ $json.workflow.name }} \nURL: {{ $json.execution.url }}"
           }
         ]
       }
       ```  
     - Replace `<UID HERE>` with your actual LINE user ID to receive messages.

5. **Set up LINE credentials in n8n:**  
   - Go to **Credentials** in n8n.  
   - Create new **HTTP Header Auth** credentials.  
   - Add the header:  
     - Key: `Authorization`  
     - Value: `Bearer {YOUR_CHANNEL_ACCESS_TOKEN}` (replace with your LINE bot's channel access token).  
   - Save credentials with a recognizable name (e.g., "Line @271dudsw MiniBear").

6. **Save and activate the workflow.**

7. **Set this workflow as the default error workflow in your n8n instance:**  
   - Follow n8n documentation: [https://docs.n8n.io/flow-logic/error-handling/](https://docs.n8n.io/flow-logic/error-handling/)  
   - This ensures that any error in any workflow triggers this error notification workflow.

8. **Test the setup:**  
   - Trigger an error in any workflow.  
   - Confirm that a notification is received on your LINE account with the workflow name and execution URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| You can set this workflow as your default error workflow to catch all errors globally in your n8n instance.   | https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow                           |
| Replace `<UID HERE>` in the HTTP Request node with your LINE user ID to receive notifications correctly.       | LINE Developers Console: https://developers.line.biz/console/                                            |
| LINE Messaging API Reference for Push API usage and message formatting.                                        | https://developers.line.biz/en/reference/messaging-api/#send-narrowcast-message                           |
| This workflow is ideal for developers, DevOps teams, business owners, and automation enthusiasts monitoring errors. | Workflow description and use cases section                                                               |

---

This documentation fully describes the workflow structure, node configurations, and setup instructions, enabling both human users and automation agents to understand, reproduce, and maintain the "n8n Error Report to Line" workflow effectively.