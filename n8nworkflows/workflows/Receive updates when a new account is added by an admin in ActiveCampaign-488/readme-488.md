Receive updates when a new account is added by an admin in ActiveCampaign

https://n8nworkflows.xyz/workflows/receive-updates-when-a-new-account-is-added-by-an-admin-in-activecampaign-488


# Receive updates when a new account is added by an admin in ActiveCampaign

### 1. Workflow Overview

This workflow is designed to monitor and receive updates whenever a new account is added by an administrator in ActiveCampaign. It is targeted at users who want to automate notifications, logging, or further processing triggered by account creation events initiated by admins within the ActiveCampaign platform.

Logical Blocks:
- **1.1 Event Listening:** The workflow begins with an event trigger node that listens specifically for new account additions performed by admins in ActiveCampaign.

---

### 2. Block-by-Block Analysis

#### 1.1 Event Listening

- **Overview:**  
  This block establishes a real-time event listener hooked into ActiveCampaign’s event system. It filters for the `account_add` event originating from admin users, effectively triggering the workflow only when an admin adds a new account.

- **Nodes Involved:**  
  - ActiveCampaign Trigger

- **Node Details:**

  **ActiveCampaign Trigger**  
  - Type and Technical Role:  
    Event trigger node specialized for ActiveCampaign events. It listens for defined event types and initiators.  
  - Configuration Choices:  
    - Events selected: `account_add` — triggers when a new account is added.  
    - Sources filtered: `admin` — ensures the event is triggered only when the account is added by an admin user.  
  - Key Expressions or Variables Used:  
    None explicitly configured; the node uses event filters provided in parameters.  
  - Input and Output Connections:  
    - This is a trigger node with no input connections.  
    - Output connection is not defined in the current workflow as no downstream nodes exist.  
  - Version-Specific Requirements:  
    Requires n8n version supporting `activeCampaignTrigger` node type version 1 or higher. Also, the ActiveCampaign API credentials must be correctly configured and authorized.  
  - Edge Cases or Potential Failure Types:  
    - Authentication errors if ActiveCampaign credentials are invalid or expired.  
    - Network timeouts or connectivity issues with ActiveCampaign API.  
    - No events triggered if the event filter is too restrictive or if no account additions occur by admins.  
  - Sub-workflow Reference:  
    None.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role            | Input Node(s) | Output Node(s) | Sticky Note                      |
|-------------------------|-----------------------------|----------------------------|----------------|----------------|---------------------------------|
| ActiveCampaign Trigger   | activeCampaignTrigger       | Listen for new account added by admin | None           | None           | None                            |

---

### 4. Reproducing the Workflow from Scratch

1. Open n8n and create a new workflow.
2. Add a new node and select **ActiveCampaign Trigger**.
3. Configure the node as follows:  
   - In the **Events** section, select `account_add`.  
   - In the **Sources** section, select `admin` to filter for account additions performed by administrators only.  
4. Set up the **ActiveCampaign API credentials**:  
   - Create or select existing credentials with valid API key and URL for your ActiveCampaign account.  
   - Assign these credentials to the ActiveCampaign Trigger node.  
5. Position the node appropriately on the canvas.
6. Save the workflow.
7. Optionally, add downstream nodes to process or notify based on the trigger (not included in this workflow).
8. Activate the workflow to start listening for events.

---

### 5. General Notes & Resources

| Note Content                                                        | Context or Link                                  |
|-------------------------------------------------------------------|-------------------------------------------------|
| ActiveCampaign event triggers require proper API credentials and permissions to access account events. | https://developers.activecampaign.com/reference/webhooks |
| This workflow currently only listens for admin-initiated account additions; expand filters to other event types as needed. | Workflow extension advice                        |

---

This documentation fully covers the workflow titled "Receive updates when a new account is added by an admin in ActiveCampaign," enabling reproduction, modification, and understanding of its operation and integration points.