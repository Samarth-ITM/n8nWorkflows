Send a message on Mattermost when a lead replies to your Lemlist email

https://n8nworkflows.xyz/workflows/send-a-message-on-mattermost-when-a-lead-replies-to-your-lemlist-email-984


# Send a message on Mattermost when a lead replies to your Lemlist email

### 1. Workflow Overview

This workflow is designed to notify a Mattermost channel whenever a lead replies to an email sent via a Lemlist campaign. It is aimed at sales, marketing, or customer success teams who want real-time updates on lead engagement without manually checking Lemlist.

The workflow consists of two main logical blocks:

- **1.1 Input Reception:** Capturing the event of a lead replying to an email in Lemlist via a webhook trigger.
- **1.2 Notification Dispatch:** Sending a formatted message containing reply details to a designated Mattermost channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming replies from leads to specific Lemlist campaigns. When a reply is detected, it triggers the workflow to continue processing.

- **Nodes Involved:**  
  - Lemlist Trigger

- **Node Details:**

  - **Lemlist Trigger**  
    - **Type and Role:**  
      Native webhook trigger node for Lemlist events, specialized in capturing email reply events.  
    - **Configuration Choices:**  
      - Event: `emailsReplied` — triggers only when a lead replies to an email.  
      - Campaign ID: `cam_H5pYEryq6mRKBiy5v` — limits trigger to replies from a specific campaign.  
    - **Expressions / Variables Used:**  
      Payload JSON includes lead details such as `firstName`, `campaignName`, and the reply `text`.  
    - **Input and Output:**  
      - Input: None (trigger node)  
      - Output: JSON data representing the reply event details.  
    - **Version Requirements:**  
      Compatible with n8n version supporting Lemlist Trigger node v1.  
    - **Potential Failures:**  
      - Authentication errors due to invalid Lemlist API credentials.  
      - Webhook registration failure.  
      - Timeout or delayed triggering if Lemlist API is slow.  
      - Missing or malformed campaign ID causing no triggers.  
    - **Sub-workflow:** None.

#### 1.2 Notification Dispatch

- **Overview:**  
  Sends a message to a specified Mattermost channel with details of the lead reply, enabling team members to be instantly informed.

- **Nodes Involved:**  
  - Mattermost

- **Node Details:**

  - **Mattermost**  
    - **Type and Role:**  
      Sends messages to Mattermost channels via Mattermost API.  
    - **Configuration Choices:**  
      - Channel ID: `qx9yo1i9z3bg5qcy5a1oxnh69c` (Leads channel) where the message is posted.  
      - Message Template:  
        ```
        {{$json["firstName"]}} has replied back to your {{$json["campaignName"]}}. Below is the reply:
        > {{$json["text"]}}
        ```  
      - Attachments: Empty array (no attachments sent).  
      - Other options: Default (none specified).  
    - **Expressions / Variables Used:**  
      Uses JSON properties from the incoming payload (`firstName`, `campaignName`, `text`) to dynamically build the message.  
    - **Input and Output:**  
      - Input: Output JSON from Lemlist Trigger node.  
      - Output: Confirmation or status from Mattermost API.  
    - **Version Requirements:**  
      Compatible with Mattermost node v1.  
    - **Potential Failures:**  
      - Authentication errors if Mattermost API credentials are invalid or expired.  
      - Incorrect channel ID causing message delivery failure.  
      - Network issues leading to API call timeouts.  
      - Malformed expressions causing message formatting errors.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name        | Node Type                 | Functional Role               | Input Node(s)     | Output Node(s)   | Sticky Note                                                                                          |
|------------------|---------------------------|------------------------------|-------------------|------------------|----------------------------------------------------------------------------------------------------|
| Lemlist Trigger  | lemlistTrigger            | Capture lead email replies    | None              | Mattermost       | The Lemlist Trigger node will trigger the workflow when a lead sends a reply to a campaign.       |
| Mattermost       | mattermost                | Send notification message     | Lemlist Trigger   | None             | This node sends a message to the `Leads` channel in Mattermost with the reply information.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Lemlist Trigger Node**  
   - Add a new node, select **Lemlist Trigger**.  
   - Set **Event** to `emailsReplied`.  
   - Under **Options**, specify the **Campaign ID** to filter replies (use your campaign’s unique ID, e.g., `cam_H5pYEryq6mRKBiy5v`).  
   - Configure **Credentials** by creating or selecting valid Lemlist API credentials with appropriate permissions.  
   - Save the node.

2. **Create Mattermost Node**  
   - Add a new node, select **Mattermost**.  
   - Set **Channel ID** to the ID of the channel where you want messages posted (e.g., `qx9yo1i9z3bg5qcy5a1oxnh69c`).  
   - In the **Message** parameter, use the following expression to compose the message dynamically:  
     ```
     {{$json["firstName"]}} has replied back to your {{$json["campaignName"]}}. Below is the reply:
     > {{$json["text"]}}
     ```  
   - Leave **Attachments** empty unless you want to add files or cards.  
   - Configure **Credentials** with valid Mattermost API credentials (typically a personal access token or bot token with posting permissions).  
   - Save the node.

3. **Connect Nodes**  
   - Link the **Lemlist Trigger** node’s output to the **Mattermost** node’s input. This ensures the notification is sent whenever a reply is detected.

4. **Activate Workflow**  
   - Ensure the workflow is active and webhook URLs are registered correctly by n8n.  
   - Test by replying to an email in the specified Lemlist campaign and verify the message appears in the Mattermost channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                      |
|-------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| To use a different notification service, replace the Mattermost node with your preferred messaging node (Slack, Teams). | Adaptability tip for other notification platforms.                 |
| Campaign ID in Lemlist Trigger must be accurate for filtering relevant replies.                                         | Lemlist dashboard to find campaign identifiers.                    |
| Mattermost channel ID is unique and can be found in Mattermost channel settings or API documentation.                   | Mattermost API docs: https://api.mattermost.com/                   |
| Ensure API credentials have appropriate permissions to avoid auth errors.                                               | Credential setup best practices in n8n documentation.              |