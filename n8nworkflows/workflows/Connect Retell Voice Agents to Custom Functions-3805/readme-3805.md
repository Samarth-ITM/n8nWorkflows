Connect Retell Voice Agents to Custom Functions

https://n8nworkflows.xyz/workflows/connect-retell-voice-agents-to-custom-functions-3805


# Connect Retell Voice Agents to Custom Functions

### 1. Workflow Overview

This workflow enables integration between Retell's Voice Agents and n8n custom logic via Retell's Custom Function webhook calls. It is designed to receive POST webhooks from Retell whenever a Voice Agent reaches a Custom Function node, process the incoming data, and respond dynamically in real-time to the agent.

**Target Use Cases:**  
- Builders using Retell who want to extend Voice Agent capabilities with custom workflows or AI-generated responses.  
- Automating actions such as booking confirmations, CRM updates, or external API calls triggered by conversational events.

**Logical Blocks:**  
- **1.1 Webhook Reception:** Receives POST requests from Retell’s Custom Function node with call context and parameters.  
- **1.2 Response Preparation:** Processes the incoming data and sets a response string to send back to the Voice Agent.  
- **1.3 Webhook Response:** Sends the prepared response back to Retell to be spoken or displayed by the Voice Agent.  
- **1.4 Documentation and Guidance (Sticky Notes):** Provides contextual instructions, usage notes, and extension ideas for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Reception

- **Overview:** Listens for incoming POST requests from Retell’s Custom Function node during a conversation. This node is the entry point of the workflow.  
- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Node Name:** Webhook  
  - **Type:** Webhook (HTTP POST endpoint)  
  - **Configuration:**  
    - Path: `hotel-retell-template` (URL endpoint for webhook calls)  
    - HTTP Method: POST  
    - Response Data: Default JSON response `{ "response": "Your booking is confirmed" }` (overridden downstream)  
  - **Input/Output:** Receives HTTP POST from Retell; outputs JSON data with call context and parameters.  
  - **Edge Cases:**  
    - Invalid or malformed POST data may cause failures.  
    - Unauthorized or unexpected requests if webhook URL is exposed.  
    - Network or timeout errors if Retell cannot reach the webhook.  
  - **Sticky Note:** "Retell Custom Function Webhook" explains this node’s role.

#### 1.2 Response Preparation

- **Overview:** Sets the response string that will be returned to Retell’s Voice Agent. This is the customizable logic block where users can replace or extend the response generation.  
- **Nodes Involved:**  
  - [Replace me!] Set response

- **Node Details:**  
  - **Node Name:** [Replace me!] Set response  
  - **Type:** Set node (assigns or modifies data fields)  
  - **Configuration:**  
    - Assigns a string value to the field `response` with the text: "Your booking has been confirmed!"  
    - This static response can be replaced with dynamic logic, API calls, or AI nodes.  
  - **Input/Output:** Receives JSON from Webhook node; outputs JSON with `response` field.  
  - **Edge Cases:**  
    - Expression errors if dynamic expressions are used improperly.  
    - Missing or incorrect data fields if downstream logic depends on input data.  
  - **Sticky Note:** "Place your logic here!" encourages users to customize this node.

#### 1.3 Webhook Response

- **Overview:** Sends the prepared response back to Retell’s Voice Agent synchronously, allowing real-time conversational feedback.  
- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Node Name:** Respond to Webhook  
  - **Type:** Respond to Webhook node  
  - **Configuration:** Default settings; sends incoming data as HTTP response.  
  - **Input/Output:** Receives JSON from Set node; outputs HTTP response to Retell.  
  - **Edge Cases:**  
    - Failure to respond in time may cause Retell to timeout.  
    - Incorrect response format may cause errors in Retell agent.  
  - **Sticky Note:** "Retell Custom Function Response" explains this node’s role.

#### 1.4 Documentation and Guidance (Sticky Notes)

- **Overview:** Provides detailed instructions, context, and usage tips embedded as sticky notes in the workflow canvas.  
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  
  - **Sticky Note1:** Full workflow overview, prerequisites, usage instructions, and extension ideas.  
  - **Sticky Note2:** Explains the webhook node role.  
  - **Sticky Note:** Encourages placing custom logic in the Set node.  
  - **Sticky Note3:** Explains the response node role.  
  - These nodes have no input or output connections; purely informational.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                      | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                              |
|--------------------------|---------------------|------------------------------------|-----------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Webhook                  | Webhook             | Receives POST webhook from Retell  | —                     | [Replace me!] Set response | "Retell Custom Function Webhook: POST Webhook received from Retell's Custom Function each time it is triggered by Retell's Voice Agent" |
| [Replace me!] Set response | Set                 | Prepares response string to return | Webhook               | Respond to Webhook     | "Place your logic here! Your Agent logic goes here. You can, for example, use an AI Agent or make an action in a third party service." |
| Respond to Webhook       | Respond to Webhook   | Sends response back to Retell      | [Replace me!] Set response | —                     | "Retell Custom Function Response: Response to the webhook that will be provided back to Retell's Voice Agent." |
| Sticky Note1             | Sticky Note         | Full workflow overview and instructions | —                     | —                     | Full detailed workflow overview, prerequisites, usage, and extension ideas                             |
| Sticky Note2             | Sticky Note         | Explains Webhook node role         | —                     | —                     | See above for Webhook explanation                                                                      |
| Sticky Note              | Sticky Note         | Encourages custom logic placement  | —                     | —                     | See above for logic placement note                                                                     |
| Sticky Note3             | Sticky Note         | Explains Respond to Webhook node   | —                     | —                     | See above for Respond to Webhook explanation                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Add a **Webhook** node.  
   - Set HTTP Method to **POST**.  
   - Set the Path to `hotel-retell-template`.  
   - Configure default response data as JSON: `{ "response": "Your booking is confirmed" }` (this will be overridden).  
   - This node will receive webhook calls from Retell’s Custom Function.

2. **Create Set Node for Response:**  
   - Add a **Set** node named `[Replace me!] Set response`.  
   - Connect the Webhook node’s output to this Set node’s input.  
   - In the Set node, add a new field:  
     - Name: `response`  
     - Type: String  
     - Value: `"Your booking has been confirmed!"`  
   - This node is where you customize the response or add additional logic (API calls, AI, etc.).

3. **Create Respond to Webhook Node:**  
   - Add a **Respond to Webhook** node.  
   - Connect the Set node’s output to this node’s input.  
   - Leave default settings to send the incoming data as the HTTP response.  
   - This node sends the response back to Retell in real-time.

4. **Connect Nodes:**  
   - Connect **Webhook → [Replace me!] Set response → Respond to Webhook** in sequence.

5. **Add Sticky Notes (Optional but Recommended):**  
   - Add sticky notes for documentation and instructions as per the original workflow for clarity.

6. **Credentials:**  
   - No external credentials are required for this basic workflow.  
   - If you extend with external API calls or AI nodes, configure the respective credentials (e.g., OpenAI API key).

7. **Deploy and Test:**  
   - Deploy the workflow.  
   - Copy the webhook URL from the Webhook node (e.g., `https://your-instance.app.n8n.cloud/webhook/hotel-retell-template`).  
   - Configure this URL in your Retell Voice Agent’s Custom Function node.  
   - Start a conversation with your Retell agent to trigger the webhook and receive the response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow allows triggering custom logic in n8n directly from Retell's Voice Agent using [Custom Functions](https://docs.retellai.com/build/conversation-flow/custom-function#custom-function). It enables real-time interaction and response customization.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Retell Custom Function Documentation                                                                     |
| Retell Agent Example available for import: a simple hotel agent calling this workflow to confirm bookings. Download link: [Retell Agent Template](https://drive.google.com/file/d/1rAcsNz-f8SyuOxO0VJ_84oPscYFpir4-/view?usp=sharing)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Retell Agent Template                                                                                     |
| How to configure your Retell Custom Function webhook URL: Edit the function node in Retell and set the webhook URL copied from n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Visual instructions included in sticky notes (images referenced)                                         |
| Extension ideas include integrating third-party APIs (hotel availability, CRM), AI-generated dynamic responses, or triggering parallel automations (Slack, calendar invites).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow extension suggestions                                                                            |
| Contact for support or conversation analysis: hello@agentstudio.io                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Support contact                                                                                           |

---

This documentation provides a complete understanding of the workflow structure, logic, and usage, enabling advanced users and automation agents to reproduce, modify, and extend the integration between Retell Voice Agents and n8n custom functions.