🛠️ Sendy Tool MCP Server 💪 6 operations

https://n8nworkflows.xyz/workflows/----sendy-tool-mcp-server----6-operations-5347


# 🛠️ Sendy Tool MCP Server 💪 6 operations

### 1. Workflow Overview

The **Sendy Tool MCP Server** workflow is designed to serve as a centralized server handling six distinct operations related to email marketing campaigns and subscriber management through the Sendy platform. It acts as a multi-command processor (MCP) that receives requests via a webhook trigger and routes them to the appropriate Sendy API operation nodes.

**Target Use Cases:**  
- Creating new email campaigns.  
- Adding subscribers to mailing lists.  
- Counting subscribers on lists.  
- Deleting subscribers.  
- Removing subscribers (likely unsubscribing or soft removal).  
- Retrieving the status of subscribers.

**Logical Blocks:**

- **1.1 Trigger Reception:** Receives incoming requests specifying which Sendy operation to perform.
- **1.2 Sendy Operations:** Six separate nodes each performing a specific Sendy API operation, activated based on the trigger input.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Reception

- **Overview:**  
  This block waits for incoming requests that specify which Sendy operation to execute. It acts as the entry point for the workflow.

- **Nodes Involved:**  
  - `Sendy Tool MCP Server`

- **Node Details:**

  - **Node Name:** Sendy Tool MCP Server  
    - **Type:** MCP Trigger (from `@n8n/n8n-nodes-langchain` package)  
    - **Technical Role:** Listens for incoming webhook requests that trigger one of multiple predefined operations.  
    - **Configuration:**  
      - Uses a webhook ID (`e593f719-a782-4e76-a04e-d09629a2d69f`) to receive external HTTP requests.  
      - No additional parameters configured within the node itself; it relies on the request payload to determine the operation.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Connects to all six Sendy Tool nodes (one-to-many).  
    - **Version-Specific Requirements:** Requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger` node type.  
    - **Potential Failures:**  
      - Webhook misconfiguration causing no trigger.  
      - Payload missing or malformed, causing downstream nodes to fail.  
      - Authentication or permission issues if webhook is secured externally.  
    - **Sub-Workflow:** None.

#### 1.2 Sendy Operations

- **Overview:**  
  This block contains six separate nodes, each invoking a specific Sendy API operation. These nodes execute the actual business logic corresponding to campaign and subscriber management.

- **Nodes Involved:**  
  - `Create a campaign`  
  - `Add a subscriber`  
  - `Count a subscriber`  
  - `Delete a subscriber`  
  - `Remove a subscriber`  
  - `Get subscriber's status`

- **Node Details:**

  - **Node Name:** Create a campaign  
    - **Type:** Sendy Tool node  
    - **Technical Role:** Creates a new email campaign in Sendy.  
    - **Configuration:** Uses Sendy API credentials and parameters (not explicitly shown) to create a campaign.  
    - **Input Connections:** Receives input from `Sendy Tool MCP Server` trigger node.  
    - **Output Connections:** None (end of branch).  
    - **Potential Failures:**  
      - API authentication or connectivity errors.  
      - Invalid campaign parameters causing API rejections.  
    - **Sub-Workflow:** None.

  - **Node Name:** Add a subscriber  
    - **Type:** Sendy Tool node  
    - **Technical Role:** Adds a subscriber to a mailing list.  
    - **Configuration:** Parameters include subscriber details and list ID (not explicitly shown).  
    - **Input Connections:** From `Sendy Tool MCP Server` trigger node.  
    - **Output Connections:** None.  
    - **Potential Failures:**  
      - Invalid subscriber data.  
      - API quota limits or authentication errors.  

  - **Node Name:** Count a subscriber  
    - **Type:** Sendy Tool node  
    - **Technical Role:** Counts the number of subscribers on a list.  
    - **Input Connections:** From `Sendy Tool MCP Server`.  
    - **Output Connections:** None.  
    - **Potential Failures:** API connectivity or invalid list ID.

  - **Node Name:** Delete a subscriber  
    - **Type:** Sendy Tool node  
    - **Technical Role:** Deletes a subscriber from a list permanently.  
    - **Input Connections:** From `Sendy Tool MCP Server`.  
    - **Output Connections:** None.  
    - **Potential Failures:** Subscriber not found, API errors.

  - **Node Name:** Remove a subscriber  
    - **Type:** Sendy Tool node  
    - **Technical Role:** Removes (likely unsubscribes) a subscriber from a list without deletion.  
    - **Input Connections:** From `Sendy Tool MCP Server`.  
    - **Output Connections:** None.  
    - **Potential Failures:** Same as delete, depending on API semantics.

  - **Node Name:** Get subscriber's status  
    - **Type:** Sendy Tool node  
    - **Technical Role:** Retrieves current subscription status of a subscriber.  
    - **Input Connections:** From `Sendy Tool MCP Server`.  
    - **Output Connections:** None.  
    - **Potential Failures:** Subscriber not found, API errors.

---

### 3. Summary Table

| Node Name              | Node Type                      | Functional Role                         | Input Node(s)          | Output Node(s)                        | Sticky Note |
|------------------------|--------------------------------|---------------------------------------|-----------------------|-------------------------------------|-------------|
| Workflow Overview 0    | Sticky Note                    | Informational placeholder             | None                  | None                                |             |
| Sendy Tool MCP Server  | MCP Trigger                    | Receives webhook, triggers operations | None                  | Create a campaign, Add a subscriber, Count a subscriber, Delete a subscriber, Remove a subscriber, Get subscriber's status |             |
| Create a campaign      | Sendy Tool                    | Creates a new email campaign           | Sendy Tool MCP Server | None                                |             |
| Sticky Note 1          | Sticky Note                    | Informational placeholder             | None                  | None                                |             |
| Add a subscriber       | Sendy Tool                    | Adds subscriber to a mailing list      | Sendy Tool MCP Server | None                                |             |
| Count a subscriber     | Sendy Tool                    | Counts subscribers on a list           | Sendy Tool MCP Server | None                                |             |
| Delete a subscriber    | Sendy Tool                    | Deletes a subscriber permanently       | Sendy Tool MCP Server | None                                |             |
| Remove a subscriber    | Sendy Tool                    | Removes/unsubscribes a subscriber      | Sendy Tool MCP Server | None                                |             |
| Get subscriber's status| Sendy Tool                    | Retrieves subscriber's subscription status | Sendy Tool MCP Server | None                            |             |
| Sticky Note 2          | Sticky Note                    | Informational placeholder             | None                  | None                                |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **MCP Trigger** node (`@n8n/n8n-nodes-langchain.mcpTrigger`).
   - Configure a webhook with a unique webhook ID or URL.
   - No additional parameters needed.
   - This node will listen for incoming requests specifying which operation to execute.

2. **Add Sendy Tool Nodes for Each Operation:**

   For each operation below, add a **Sendy Tool** node and configure it according to the Sendy API and your credentials.

   - **Create a campaign:**
     - Node name: `Create a campaign`
     - Configure with Sendy API credentials.
     - Set parameters for campaign creation (e.g., campaign title, content, list ID).
     - Connect input to `Sendy Tool MCP Server`.

   - **Add a subscriber:**
     - Node name: `Add a subscriber`
     - Configure with Sendy API credentials.
     - Set subscriber details (email, name, list ID).
     - Connect input to `Sendy Tool MCP Server`.

   - **Count a subscriber:**
     - Node name: `Count a subscriber`
     - Configure with Sendy API credentials.
     - Set list ID to count subscribers.
     - Connect input to `Sendy Tool MCP Server`.

   - **Delete a subscriber:**
     - Node name: `Delete a subscriber`
     - Configure with Sendy API credentials.
     - Provide subscriber email and list ID.
     - Connect input to `Sendy Tool MCP Server`.

   - **Remove a subscriber:**
     - Node name: `Remove a subscriber`
     - Configure similarly to Delete but set to unsubscribe or remove softly.
     - Connect input to `Sendy Tool MCP Server`.

   - **Get subscriber's status:**
     - Node name: `Get subscriber's status`
     - Configure with subscriber email and list ID.
     - Connect input to `Sendy Tool MCP Server`.

3. **Connect the Nodes:**
   - From the **MCP Trigger** node, connect its `ai_tool` output (or main output) to the input of all six Sendy Tool nodes.  
   - Each Sendy Tool node acts independently based on the incoming request content.

4. **Set Credentials:**
   - For each Sendy Tool node, configure the Sendy API credentials with the necessary API key and base URL.
   - Ensure credentials have permissions for the respective operations.

5. **Additional Configurations:**
   - Optionally, add error handling or branching to handle invalid commands or errors.
   - Add sticky notes or comments for clarity.

6. **Activate the Workflow:**
   - Save and activate the workflow.
   - Use the webhook URL from the MCP Trigger as the endpoint to send commands specifying which operation to perform along with required parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                          |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow integrates with Sendy, a self-hosted email marketing application. Ensure your Sendy instance is accessible and API enabled. | Sendy official site: https://sendy.co  |
| The MCP Trigger node enables multi-command processing, simplifying the workflow by centralizing webhook handling. | n8n MCP Trigger documentation           |
| To test API operations, use tools like Postman or curl to send JSON requests to the webhook URL.                | n8n webhook testing instructions       |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.