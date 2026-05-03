Update Shopify order tags when a Onfleet event happens

https://n8nworkflows.xyz/workflows/update-shopify-order-tags-when-a-onfleet-event-happens-1545


# Update Shopify order tags when a Onfleet event happens

### 1. Workflow Overview

This workflow automates the synchronization between Onfleet delivery events and Shopify order management by updating Shopify order tags whenever specific Onfleet events occur. It is designed for merchants using Shopify for e-commerce and Onfleet for last-mile delivery management to keep order tags in Shopify reflective of real-time delivery statuses or events from Onfleet.

Logical blocks included:

- **1.1 Event Listening:** Listens for a specific event occurring in Onfleet (e.g., a task delay).
- **1.2 Shopify Order Update:** Upon receiving the event, updates the tags of the corresponding Shopify order.

---

### 2. Block-by-Block Analysis

#### 2.1 Onfleet Event Listening Block

- **Overview:**  
  This block is responsible for receiving trigger events from Onfleet. It listens specifically for the "taskDelayed" event, which occurs when a delivery task is delayed.

- **Nodes Involved:**  
  - Onfleet Trigger

- **Node Details:**  

  **Onfleet Trigger**  
  - Type: Trigger node specific to Onfleet webhook events.  
  - Configuration:  
    - Trigger on: `taskDelayed` event (can be changed to listen to other Onfleet events).  
    - Webhook ID: Unique identifier for the webhook endpoint created in n8n.  
    - Additional fields: None configured.  
  - Key Expressions/Variables: Not applicable as it listens passively for the event.  
  - Input: External HTTP webhook invocation by Onfleet when the specified event occurs.  
  - Output: Passes event data downstream to the Shopify node.  
  - Version-specific requirements: Uses typeVersion 1 of the node; ensure n8n version supports this node type.  
  - Potential failure points:  
    - Incorrect or expired Onfleet API key or webhook misconfiguration could prevent event reception.  
    - Network or webhook endpoint downtime could cause missed events.  
    - Event payload changes by Onfleet API could cause parsing issues downstream.  
  - Sub-workflow: None.

#### 2.2 Shopify Order Tag Update Block

- **Overview:**  
  This block updates the Shopify order tags based on the Onfleet event data received. It modifies the tags field of the Shopify order to reflect the delivery status or other custom tagging logic.

- **Nodes Involved:**  
  - Shopify

- **Node Details:**  

  **Shopify**  
  - Type: Integration node for Shopify API.  
  - Configuration:  
    - Operation: `update` (update an existing Shopify order).  
    - Update Fields: Configured to update the `tags` field. The tags value is set to an empty string by default in this template, meaning it requires customization to specify what tags to apply based on the event.  
  - Key Expressions/Variables: None preset; users should define expressions or static values for tags based on Onfleet event data.  
  - Input: Receives event data from the Onfleet Trigger node.  
  - Output: Outputs the response from Shopify API after updating the order tags.  
  - Version-specific requirements: Uses typeVersion 1 of the node. Ensure Shopify API credentials are valid and have permissions to update orders.  
  - Potential failure points:  
    - Missing or invalid Shopify credentials will cause authentication errors.  
    - Incorrect order ID or data mapping may fail the update operation.  
    - API rate limits or network issues could cause timeouts or failures.  
  - Sub-workflow: None.

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role            | Input Node(s)   | Output Node(s) | Sticky Note                                                                                          |
|-----------------|---------------------|----------------------------|-----------------|----------------|----------------------------------------------------------------------------------------------------|
| Onfleet Trigger | on8n-nodes-base.onfleetTrigger | Event listener for Onfleet delivery events | -               | Shopify        | Update the Onfleet trigger node with your own Onfleet credentials, register for an Onfleet API key at https://onfleet.com/signup |
| Shopify         | n8n-nodes-base.shopify | Updates Shopify order tags based on event | Onfleet Trigger | -              | Update the Shopify node with your Shopify credentials and add your own tags to the Shopify Order  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Onfleet Trigger Node**  
   - Node Type: Onfleet Trigger  
   - Position: (e.g., x=460, y=300) for clarity  
   - Configuration:  
     - Select the trigger event type to listen for, e.g., `taskDelayed` (or other events as needed).  
     - Ensure you have valid Onfleet credentials (API key).  
     - The node automatically sets up a webhook URL—note this URL to configure Onfleet webhook if required.  

2. **Create Shopify Node**  
   - Node Type: Shopify  
   - Position: (e.g., x=680, y=300)  
   - Configuration:  
     - Operation: `update` (to update an existing order).  
     - Update Fields: Set the `tags` field to the desired tag(s). For dynamic tagging, use expressions referencing data from Onfleet Trigger node (e.g., `{{$json["some_field"]}}`).  
     - Credentials: Connect with your Shopify store OAuth2 or API credentials ensuring permissions to update orders.  
     - Map the order ID or other identifiers from the Onfleet event payload to the Shopify order to update (this step requires adding an expression or manual mapping for order identification).  

3. **Connect Nodes**  
   - Link the output of the Onfleet Trigger node to the input of the Shopify node.

4. **Activate Workflow**  
   - Save and activate the workflow to start listening for Onfleet events and updating Shopify tags accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To register for an Onfleet API key, visit https://onfleet.com/signup                                           | Onfleet API Registration                                                                        |
| Learn more about Onfleet webhooks and available events at https://support.onfleet.com/hc/en-us/articles/360045763852-Webhooks | Onfleet webhook documentation                                                                   |
| Ensure Shopify credentials used have appropriate permissions for order updates                                 | Shopify API permissions                                                                         |
| Customize tag update logic in Shopify node to reflect specific business needs (e.g., based on event details)  | Workflow customization                                                                          |

---

This documentation facilitates understanding, reproducing, and customizing the workflow for syncing Onfleet delivery events with Shopify order tags, enabling smooth integration and operational efficiency.