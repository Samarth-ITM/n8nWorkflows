Send a message on Mattermost when you get a reply in Emelia

https://n8nworkflows.xyz/workflows/send-a-message-on-mattermost-when-you-get-a-reply-in-emelia-1039


# Send a message on Mattermost when you get a reply in Emelia

### 1. Workflow Overview

This workflow automates notifications by sending a message to a Mattermost channel whenever a lead replies to an email campaign managed via Emelia. It is designed for sales or marketing teams who want immediate alerts about lead engagement without manually monitoring email replies.

The workflow is logically divided into two main blocks:

- **1.1 Input Reception:** Listens for reply events from Emelia campaigns using a webhook trigger.
- **1.2 Notification Dispatch:** Sends a customized message to a designated Mattermost channel informing the team about the lead’s reply.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block activates the workflow when a recipient replies to a specific Emelia campaign email. It acts as the event source for downstream processes.

**Nodes Involved:**  
- Emelia Trigger

**Node Details:**  

- **Name:** Emelia Trigger  
- **Type:** Emelia Trigger (Webhook Event Listener)  
- **Technical Role:** Listens for specific events ("replied") from Emelia campaigns via webhook and outputs data about the reply and contact.  
- **Configuration Choices:**  
  - Trigger on the event type: “replied” (indicates a lead replied to the campaign)  
  - Campaign ID specified: `6054d068b374b64365740101` (limits triggers to replies from this campaign only)  
  - Webhook ID: a fixed UUID to receive callbacks from Emelia  
- **Key Expressions/Variables:**  
  - Outputs JSON containing lead contact details and reply metadata (e.g., `$json["contact"]["firstName"]`, `$json["contact"]["company"]`)  
- **Input Connections:** None (entry point)  
- **Output Connections:** Connected to the Mattermost node to forward event data  
- **Version-specific Requirements:** Requires Emelia API credentials configured and webhook properly registered with Emelia (API version compatibility)  
- **Potential Failures:**  
  - Authentication errors with Emelia API credentials  
  - Webhook registration or connectivity issues (if Emelia fails to call webhook)  
  - Campaign ID mismatch or invalid configuration causing no triggers  
- **Sub-workflow Reference:** None  

#### 1.2 Notification Dispatch

**Overview:**  
This block sends a notification message to a specified Mattermost channel informing the team about the lead's reply, using data from the Emelia Trigger.

**Nodes Involved:**  
- Mattermost

**Node Details:**  

- **Name:** Mattermost  
- **Type:** Mattermost node (Message sender)  
- **Technical Role:** Posts a message into a Mattermost channel with contextual information about the reply event  
- **Configuration Choices:**  
  - Message template: `"{{$json["contact"]["firstName"]}} from {{$json["contact"]["company"]}} has replied back to your campaign."`  
  - Target channel ID: `qx9yo1i9z3bg5qcy5a1oxnh69c` (corresponds to the "Leads" channel)  
  - No attachments or other options specified  
- **Key Expressions/Variables:**  
  - Uses JSON references from the input data to personalize the message  
- **Input Connections:** Receives data from Emelia Trigger node  
- **Output Connections:** None (endpoint)  
- **Version-specific Requirements:** Requires properly configured Mattermost API credentials and access rights to post to the specified channel  
- **Potential Failures:**  
  - Authentication errors with Mattermost API credentials  
  - Invalid or expired channel ID  
  - Message formatting errors if input JSON missing expected fields  
- **Sub-workflow Reference:** None  

---

### 3. Summary Table

| Node Name     | Node Type                  | Functional Role                  | Input Node(s)  | Output Node(s) | Sticky Note                                                                                      |
|---------------|----------------------------|--------------------------------|----------------|----------------|------------------------------------------------------------------------------------------------|
| Emelia Trigger| Emelia Trigger             | Listens for lead reply events  | -              | Mattermost     | The Emelia Trigger node will trigger the workflow when a lead sends a reply to a campaign      |
| Mattermost    | Mattermost                 | Sends notification message     | Emelia Trigger | -              | This node will send a message to the `Leads` channel in Mattermost with the information about the reply. Based on your use case, you may want to send the message to a different channel. You may even want to use a different service. Replace the node with the service where you want to send a message. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Emelia Trigger Node**  
   - Add a new node of type “Emelia Trigger”.  
   - Configure the node to listen for the event type: `replied`.  
   - Set the Campaign ID to `6054d068b374b64365740101`.  
   - Use or create Emelia API credentials: provide API key/token and ensure webhook setup is complete on Emelia’s side, with the webhook URL pointing to this n8n instance.  
   - Save the node.

2. **Create Mattermost Node**  
   - Add a new node of type “Mattermost”.  
   - Configure the node to send a message to the channel with ID `qx9yo1i9z3bg5qcy5a1oxnh69c` (this corresponds to the “Leads” channel).  
   - Enter the message template exactly as:  
     `={{$json["contact"]["firstName"]}} from {{$json["contact"]["company"]}} has replied back to your campaign.`  
   - Use or create Mattermost API credentials with permission to post messages to the specified channel.  
   - Save the node.

3. **Connect Nodes**  
   - Connect the output of the “Emelia Trigger” node to the input of the “Mattermost” node directly, ensuring data flows from trigger to notification.

4. **Activate Credentials and Webhook**  
   - Verify that Emelia API credentials are valid and the webhook is registered on Emelia’s platform.  
   - Verify Mattermost credentials allow posting to the chosen channel.  
   - Activate the workflow and test by replying to the specified Emelia campaign to confirm message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow is designed specifically to notify a Mattermost channel named “Leads” when a reply is received from a campaign lead in Emelia.            | Workflow purpose                                                                                 |
| You may customize the target Mattermost channel or replace the node with another messaging service node to fit different notification needs.             | Customization advice                                                                            |
| Ensure that the Emelia webhook is correctly registered and reachable from the n8n instance to guarantee trigger reliability.                             | Emelia webhook setup requirement                                                               |
| For more information about Emelia Triggers, see the official Emelia API and n8n documentation.                                                           | https://docs.emelia.io/ and https://docs.n8n.io/                                                |
| For Mattermost API credential setup and channel information, consult Mattermost developer docs.                                                          | https://developers.mattermost.com/integrate/apis/                                              |

---

This documentation fully describes the workflow structure, configuration, and operational details to enable reproduction, maintenance, and extension.