🧠 *NEW* Claude 3.7 Extended Thinking AI Agent 🤖 – Unlock Ultimate Intelligence

https://n8nworkflows.xyz/workflows/----new--claude-3-7-extended-thinking-ai-agent------unlock-ultimate-intelligence-3009


# 🧠 *NEW* Claude 3.7 Extended Thinking AI Agent 🤖 – Unlock Ultimate Intelligence

### 1. Workflow Overview

This workflow, titled **"Claude 3.7 Extended Thinking AI Agent"**, is designed to enable advanced users to interact with the Claude 3.7 AI model via its API, unlocking enhanced customization and deeper reasoning capabilities beyond the official interface limits. It targets AI enthusiasts, researchers, professionals, analysts, and developers who want to push Claude’s intelligence further by adjusting parameters such as thinking tokens and other settings.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures user input via a form submission trigger.
- **1.2 AI Processing:** Sends the user input to the Claude 3.7 API with customized parameters and retrieves the AI’s response.
- **1.3 Output Handling:** (Implicit in the workflow) Forwards the AI response to the next node or system (not explicitly shown but implied by node connections).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for user input submitted through a web form and triggers the workflow execution upon submission.

- **Nodes Involved:**  
  - On form submission  
  - Form

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point that activates the workflow when a user submits a form.  
    - Configuration: Uses a webhook ID to listen for incoming form submissions. No additional parameters configured.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Claude Message" node  
    - Edge cases: Possible webhook connectivity issues, form submission errors, or missing required fields in the form data.  
    - Version: 2.2

  - **Form**  
    - Type: Form node  
    - Role: Represents the form data structure and captures user inputs for processing.  
    - Configuration: Uses a webhook ID to receive form data. No explicit parameters set in the JSON, implying default form behavior.  
    - Inputs: Receives output from "Claude Message" node (though connection is shown from "Claude Message" to "Form" which may be a misconfiguration or placeholder)  
    - Outputs: None (no downstream nodes connected)  
    - Edge cases: Form validation errors, missing or malformed input data.  
    - Version: 1

#### 1.2 AI Processing

- **Overview:**  
  This block sends the user’s input to the Claude 3.7 API with extended customization options, including increased thinking tokens for deeper reasoning, and retrieves the AI-generated response.

- **Nodes Involved:**  
  - Claude Message

- **Node Details:**

  - **Claude Message**  
    - Type: HTTP Request  
    - Role: Sends a POST request to the Claude 3.7 API endpoint with customized parameters and user input, then receives the AI response.  
    - Configuration:  
      - HTTP method: POST  
      - URL: Configured to Claude 3.7 API endpoint (not explicitly shown in JSON)  
      - Authentication: Uses API credentials (Claude API key) configured in n8n credentials (not shown in JSON but required)  
      - Headers: Likely includes authorization and content-type headers (application/json)  
      - Body: Contains user input and extended settings such as increased thinking tokens, temperature, max tokens, etc. (details not visible in JSON but implied by workflow description)  
    - Expressions/Variables: Uses incoming form data from "On form submission" node as input payload.  
    - Inputs: Receives data from "On form submission" node  
    - Outputs: Sends output to "Form" node (though this connection seems reversed or unused)  
    - Edge cases: API authentication failures, rate limiting, network timeouts, invalid parameter errors, malformed JSON payloads, or unexpected API responses.  
    - Version: 4.2

#### 1.3 Output Handling

- **Overview:**  
  Although not explicitly detailed in the workflow JSON, the output from the Claude API would typically be processed or forwarded for display or further use.

- **Nodes Involved:**  
  - None explicitly connected in this workflow.

- **Node Details:**  
  No nodes are configured to handle or display the AI response explicitly in this workflow. This suggests the workflow is either incomplete or designed to be extended by the user.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role               | Input Node(s)       | Output Node(s)     | Sticky Note |
|--------------------|---------------------|------------------------------|---------------------|--------------------|-------------|
| On form submission  | Form Trigger        | Trigger workflow on form submit | None                | Claude Message     |             |
| Claude Message     | HTTP Request        | Send input to Claude 3.7 API and get response | On form submission | Form               |             |
| Form               | Form                | Capture and represent user input | Claude Message      | None               |             |
| Sticky Note2       | Sticky Note         | (Empty content)               | None                | None               |             |
| Sticky Note        | Sticky Note         | (Empty content)               | None                | None               |             |
| Sticky Note1       | Sticky Note         | (Empty content)               | None                | None               |             |
| Sticky Note3       | Sticky Note         | (Empty content)               | None                | None               |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**  
   - Add a **Form Trigger** node.  
   - Configure it with a webhook ID (auto-generated by n8n).  
   - This node will listen for incoming form submissions to trigger the workflow.

2. **Create the "Claude Message" node:**  
   - Add an **HTTP Request** node.  
   - Set the HTTP Method to **POST**.  
   - Set the URL to the Claude 3.7 API endpoint (e.g., `https://api.anthropic.com/v1/claude-3.7/complete` or the correct endpoint per Claude API docs).  
   - Under Authentication, select or create credentials for the **Claude API key** (ensure you have a valid paid API key).  
   - Set Headers to include:  
     - `Content-Type: application/json`  
     - `Authorization: Bearer <your_api_key>` (if required)  
   - In the Body Parameters, configure JSON to include:  
     - The user input text (from the form submission node) mapped into the prompt or input field.  
     - Extended settings such as increased thinking tokens, temperature, max tokens, or other Claude-specific parameters to enhance reasoning depth.  
   - Connect the **On form submission** node output to this node’s input.

3. **Create the "Form" node:**  
   - Add a **Form** node to represent the form structure (optional if you want to define or validate form fields).  
   - Configure the form fields as needed to capture user input.  
   - Connect the output of the **Claude Message** node to this node if you want to process or display the response (though in the current workflow, this connection is present but not functionally used).

4. **Connect nodes:**  
   - Connect **On form submission** → **Claude Message** → **Form** (optional).  
   - Ensure the webhook URLs are active and accessible.

5. **Activate the workflow:**  
   - Save and activate the workflow.  
   - Test by submitting the form to trigger the workflow and receive AI responses.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow unlocks full customization of Claude 3.7’s AI parameters, including thinking tokens for deeper reasoning. | Workflow description and purpose.                                                               |
| Requires a paid Claude 3.7 API key for operation.                                             | API credentials setup in n8n.                                                                   |
| The workflow is designed for AI enthusiasts, researchers, professionals, and developers seeking enhanced AI interaction. | Target audience.                                                                                |
| For more information on Claude API usage and parameters, refer to Anthropic’s official API docs. | https://www.anthropic.com/index/api                                                               |
| The workflow currently lacks explicit output handling or response display nodes; users should extend it as needed. | Suggested enhancement for practical use.                                                        |

---

This documentation provides a complete understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.