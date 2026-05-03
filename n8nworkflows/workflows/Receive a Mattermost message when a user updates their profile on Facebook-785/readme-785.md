Receive a Mattermost message when a user updates their profile on Facebook

https://n8nworkflows.xyz/workflows/receive-a-mattermost-message-when-a-user-updates-their-profile-on-facebook-785


# Receive a Mattermost message when a user updates their profile on Facebook

### 1. Workflow Overview

This workflow monitors user profile updates on Facebook and sends a notification message to a specified Mattermost channel when such an update occurs. It is designed for teams or organizations that want to track changes in Facebook user profiles in real-time and receive immediate alerts within their Mattermost communication platform.

Logical blocks:

- **1.1 Facebook Profile Update Trigger**: Listens for Facebook user profile changes using Facebook’s webhook system.
- **1.2 Mattermost Notification**: Sends a formatted message to a Mattermost channel with details about the profile update.

---

### 2. Block-by-Block Analysis

#### 1.1 Facebook Profile Update Trigger

- **Overview:**  
  This block acts as the event listener that triggers the workflow whenever a user updates their Facebook profile. It relies on Facebook’s webhook infrastructure to notify n8n of changes.

- **Nodes Involved:**  
  - Facebook Trigger

- **Node Details:**

  - **Node Name:** Facebook Trigger  
  - **Type:** Facebook Trigger (Webhook / Event Listener)  
  - **Technical Role:** Listens for Facebook Graph API webhook events indicating user profile updates.  
  - **Configuration Choices:**  
    - `appId`: Not specified in JSON (must be configured with the Facebook App ID in credentials).  
    - `options.includeValues`: Set to true, meaning the payload includes detailed values of changes.  
  - **Key Expressions/Variables:**  
    - Outputs a JSON object including fields such as `uid` (User ID), `changes` array containing the updated field, and the new value under `value`.  
  - **Input/Output Connections:**  
    - No input connections (entry point node).  
    - Output connected to the Mattermost node.  
  - **Version Requirements:** Compatible with n8n version supporting Facebook Trigger node version 1.  
  - **Potential Failure Types:**  
    - Authentication errors if Facebook credentials are invalid or expired.  
    - Webhook registration failures if the Facebook app is misconfigured.  
    - Missing or malformed payload if Facebook sends unexpected data.  
  - **Sub-workflow:** None.

#### 1.2 Mattermost Notification

- **Overview:**  
  Sends a notification message to a specific Mattermost channel containing details about the Facebook profile update event received.

- **Nodes Involved:**  
  - Mattermost

- **Node Details:**

  - **Node Name:** Mattermost  
  - **Type:** Mattermost node (API integration node)  
  - **Technical Role:** Sends messages to a Mattermost channel via API.  
  - **Configuration Choices:**  
    - `message`: A dynamic expression constructed from Facebook Trigger output:  
      `"The user with uid {{$node["Facebook Trigger"].json["uid"]}} changed their {{$node["Facebook Trigger"].json["changes"][0]["field"]}} to {{$node["Facebook Trigger"].json["changes"][0]["value"]["page"]}}."`  
      This message reports the user ID, the changed field, and the new value (specifically the `page` subfield of the value object).  
    - `channelId`: Set to `"13fx8838gtbj3d41a6a7c1w7fe"`, identifying the Mattermost channel where the message will be posted.  
    - `attachments`: Empty array, no attachments sent.  
    - `otherOptions`: Empty object, no additional options configured.  
  - **Key Expressions/Variables:** Uses data from the Facebook Trigger node for message population.  
  - **Input/Output Connections:**  
    - Input connected from Facebook Trigger node.  
    - No further output connections.  
  - **Version Requirements:** Uses Mattermost node version 1, requires valid Mattermost API credentials.  
  - **Potential Failure Types:**  
    - Authentication errors if Mattermost credentials are invalid or expired.  
    - API rate limits or connectivity issues with Mattermost server.  
    - Message formatting errors if the dynamic data is missing or malformed.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role                 | Input Node(s)    | Output Node(s) | Sticky Note                                                                                                  |
|------------------|---------------------|--------------------------------|------------------|----------------|--------------------------------------------------------------------------------------------------------------|
| Facebook Trigger | Facebook Trigger    | Detect Facebook profile updates | None (Trigger)   | Mattermost     |                                                                                                              |
| Mattermost       | Mattermost          | Send notification to Mattermost | Facebook Trigger | None           |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Facebook Trigger Node**  
   - Add a new node of type **Facebook Trigger**.  
   - Configure credentials by selecting or creating Facebook Graph API credentials with appropriate app permissions for webhook events.  
   - Set **App ID** if required (in credentials).  
   - Enable `includeValues` option to true to get detailed change values.  
   - Position the node as the entry trigger.

2. **Create Mattermost Node**  
   - Add a new node of type **Mattermost**.  
   - Configure credentials with valid Mattermost API credentials (token and server URL).  
   - Set the **Channel ID** to `"13fx8838gtbj3d41a6a7c1w7fe"` or your target channel's ID.  
   - Set the **Message** parameter as the following expression:  
     ```
     The user with uid {{$node["Facebook Trigger"].json["uid"]}} changed their {{$node["Facebook Trigger"].json["changes"][0]["field"]}} to {{$node["Facebook Trigger"].json["changes"][0]["value"]["page"]}}.
     ```  
   - Leave Attachments and Other Options empty.

3. **Connect Nodes**  
   - Connect the output of the Facebook Trigger node to the input of the Mattermost node.

4. **Activate Workflow**  
   - Save and activate the workflow.  
   - Ensure the Facebook app has configured webhook subscriptions pointing to the n8n webhook URL for this Facebook Trigger.  
   - Test by updating a Facebook user's profile to verify Mattermost message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| The workflow screenshot is referenced as `fileId:310` but not included here.                     | Visual overview of workflow layout                             |
| Facebook app must have webhook subscriptions configured for user profile updates to trigger this workflow. | Facebook Developers portal webhook setup                       |
| Mattermost channel ID must be valid and the bot token must have permission to post messages.    | Mattermost API documentation                                   |