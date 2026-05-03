Create, Update Alerts 🛠️ UptimeRobot Tool MCP Server 💪 all 21 operations

https://n8nworkflows.xyz/workflows/create--update-alerts-----uptimerobot-tool-mcp-server----all-21-operations-5075


# Create, Update Alerts 🛠️ UptimeRobot Tool MCP Server 💪 all 21 operations

### 1. Workflow Overview

This workflow automates all 21 primary operations related to managing UptimeRobot resources via the UptimeRobot Tool MCP Server integration node in n8n. Its main purpose is to enable creation, retrieval, updating, and deletion of key UptimeRobot entities such as accounts, alert contacts, maintenance windows, monitors, and public status pages. The workflow is triggered by the MCP Server webhook node that acts as an entry point to route requests to the appropriate UptimeRobot API operation nodes.

The logical structure can be divided into the following blocks based on resource types and their operations:

- **1.1 Trigger and Input Reception**  
  Entry point via MCP Server trigger node that receives external requests.

- **1.2 Account Operations**  
  Single node to get account information.

- **1.3 Alert Contact Operations**  
  Nodes to create, retrieve (one or many), update, and delete alert contacts.

- **1.4 Maintenance Window Operations**  
  Nodes to create, retrieve (one or many), update, and delete maintenance windows.

- **1.5 Monitor Operations**  
  Nodes to create, retrieve (one or many), update, delete, and reset monitors.

- **1.6 Public Status Page Operations**  
  Nodes to create, retrieve (one or many), and delete public status pages.

Each operation node is connected as a child of the MCP Server trigger node, which routes the incoming requests to the correct operation based on the input parameters.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Reception

**Overview:**  
This block contains the main entry point node that listens for incoming MCP Server events or API calls to initiate the workflow and route the requests.

**Nodes Involved:**  
- UptimeRobot Tool MCP Server

**Node Details:**  
- **Name:** UptimeRobot Tool MCP Server  
- **Type:** MCP Trigger Node (n8n-nodes-langchain.mcpTrigger)  
- **Role:** Entry point for the workflow; triggers workflow execution upon receiving HTTP webhook calls with MCP Server events or commands.  
- **Configuration:**  
  - Uses a webhook URL with a generated webhookId to receive requests.  
  - No parameters configured inside the node itself; acts as a dispatcher.  
- **Inputs:** None (trigger node).  
- **Outputs:** Multiple outputs connected to all UptimeRobot operation nodes.  
- **Version Requirements:** n8n version supporting MCP Trigger node (usually v0.194.0+).  
- **Edge Cases / Failure Types:**  
  - Webhook URL not accessible or misconfigured could prevent triggering.  
  - Invalid or malformed requests may cause routing failures.  
- **Sub-workflow:** None.

---

#### 1.2 Account Operations

**Overview:**  
Retrieves the UptimeRobot account information for the authenticated user.

**Nodes Involved:**  
- Get an account

**Node Details:**  
- **Name:** Get an account  
- **Type:** UptimeRobot Tool Node (n8n-nodes-base.uptimeRobotTool)  
- **Role:** Fetch account details such as account name, email, and settings.  
- **Configuration:**  
  - Operation set to "Get an account".  
  - Requires UptimeRobot credentials (API key).  
- **Inputs:** Connected from MCP Server trigger node output.  
- **Outputs:** Delivers account data downstream or as a response.  
- **Version Requirements:** Compatible with UptimeRobot API v2+.  
- **Edge Cases / Failure Types:**  
  - Authentication errors if API key invalid or revoked.  
  - API rate limits causing request throttling.  
- **Sub-workflow:** None.

---

#### 1.3 Alert Contact Operations

**Overview:**  
Manages alert contacts used by UptimeRobot to send notifications.

**Nodes Involved:**  
- Create an alert contact  
- Delete an alert contact  
- Get an alert contact  
- Get many alert contacts  
- Update an alert contact

**Node Details for each:**

1. **Create an alert contact**  
   - Creates a new alert contact with specified contact details and types.  
   - Requires parameters like contact type (email, SMS, etc.), value, friendly name.

2. **Delete an alert contact**  
   - Deletes an existing alert contact by ID.

3. **Get an alert contact**  
   - Retrieves details of a specific alert contact by ID.

4. **Get many alert contacts**  
   - Retrieves a list of all alert contacts for the account.

5. **Update an alert contact**  
   - Updates properties (name, contact info, types) of an existing alert contact.

**Common Configuration:**  
- Each node sets the appropriate operation in UptimeRobot Tool node parameters.  
- Requires UptimeRobot API credentials.  
- Input connections from MCP Server trigger node.  
- Outputs deliver data or confirmation of operation success.

**Edge Cases / Failure Types:**  
- Trying to delete or update non-existent contact IDs.  
- Invalid contact details causing validation errors.  
- API rate limits or authentication failures.

---

#### 1.4 Maintenance Window Operations

**Overview:**  
Handles scheduling and managing maintenance windows for monitors.

**Nodes Involved:**  
- Create a maintenance window  
- Delete a maintenance window  
- Get a maintenance window  
- Get many maintenance windows  
- Update a maintenance window

**Node Details:**  
- Similar structure to alert contacts block, nodes provide CRUD operations for maintenance windows.  
- Parameters include monitor IDs, start/end times, friendly names, and descriptions.  
- Connected to MCP Server trigger node.  
- Requires API credentials.

**Edge Cases / Failure Types:**  
- Time window conflicts or invalid date formats.  
- Attempting operations on non-existent maintenance windows.  
- API authentication or rate limiting.

---

#### 1.5 Monitor Operations

**Overview:**  
Manages monitors that track uptime or performance of services.

**Nodes Involved:**  
- Create a monitor  
- Delete a monitor  
- Get a monitor  
- Get many monitors  
- Reset a monitor  
- Update a monitor

**Node Details:**  
- Provides full lifecycle management of monitors.  
- Includes reset operation to restart monitor statistics.  
- Parameters include monitor type (HTTP, ping), URL/IP, interval, and alert contacts.  
- Connected to MCP Server trigger node; requires API key.

**Edge Cases / Failure Types:**  
- Invalid monitor configuration (e.g., unsupported types or invalid URLs).  
- Resetting monitors that do not exist.  
- Permission or authentication errors.

---

#### 1.6 Public Status Page Operations

**Overview:**  
Manages public-facing status pages showing monitor statuses.

**Nodes Involved:**  
- Create a public status page  
- Delete a public status page  
- Get a public status page  
- Get many public status pages

**Node Details:**  
- Enables creation and management of status pages with configurable monitors and branding.  
- Parameters include status page title, monitors included, and visibility settings.  
- Connected to MCP Server trigger node; requires credentials.

**Edge Cases / Failure Types:**  
- Attempting to delete or update non-existent status pages.  
- Misconfiguration causing inaccessible pages.  
- API errors or rate-limiting.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                | Input Node(s)               | Output Node(s)              | Sticky Note                           |
|-------------------------------|--------------------------------|-------------------------------|-----------------------------|-----------------------------|-------------------------------------|
| Workflow Overview 0            | Sticky Note                   | Documentation placeholder      |                             |                             |                                     |
| UptimeRobot Tool MCP Server    | MCP Trigger                   | Main workflow trigger          |                             | All UptimeRobot operation nodes |                                     |
| Get an account                 | UptimeRobot Tool              | Retrieve account info          | UptimeRobot Tool MCP Server |                             |                                     |
| Sticky Note 1                 | Sticky Note                   | Documentation placeholder      |                             |                             |                                     |
| Create an alert contact        | UptimeRobot Tool              | Create alert contact           | UptimeRobot Tool MCP Server |                             |                                     |
| Delete an alert contact        | UptimeRobot Tool              | Delete alert contact           | UptimeRobot Tool MCP Server |                             |                                     |
| Get an alert contact           | UptimeRobot Tool              | Get alert contact details      | UptimeRobot Tool MCP Server |                             |                                     |
| Get many alert contacts        | UptimeRobot Tool              | List all alert contacts        | UptimeRobot Tool MCP Server |                             |                                     |
| Update an alert contact        | UptimeRobot Tool              | Update alert contact           | UptimeRobot Tool MCP Server |                             |                                     |
| Sticky Note 2                 | Sticky Note                   | Documentation placeholder      |                             |                             |                                     |
| Create a maintenance window    | UptimeRobot Tool              | Create maintenance window      | UptimeRobot Tool MCP Server |                             |                                     |
| Delete a maintenance window    | UptimeRobot Tool              | Delete maintenance window      | UptimeRobot Tool MCP Server |                             |                                     |
| Get a maintenance window       | UptimeRobot Tool              | Retrieve maintenance window    | UptimeRobot Tool MCP Server |                             |                                     |
| Get many maintenance windows   | UptimeRobot Tool              | List all maintenance windows   | UptimeRobot Tool MCP Server |                             |                                     |
| Update a maintenance window    | UptimeRobot Tool              | Update maintenance window      | UptimeRobot Tool MCP Server |                             |                                     |
| Sticky Note 3                 | Sticky Note                   | Documentation placeholder      |                             |                             |                                     |
| Create a monitor               | UptimeRobot Tool              | Create uptime monitor          | UptimeRobot Tool MCP Server |                             |                                     |
| Delete a monitor              | UptimeRobot Tool              | Delete uptime monitor          | UptimeRobot Tool MCP Server |                             |                                     |
| Get a monitor                 | UptimeRobot Tool              | Retrieve monitor details       | UptimeRobot Tool MCP Server |                             |                                     |
| Get many monitors             | UptimeRobot Tool              | List all monitors              | UptimeRobot Tool MCP Server |                             |                                     |
| Reset a monitor              | UptimeRobot Tool              | Reset monitor statistics       | UptimeRobot Tool MCP Server |                             |                                     |
| Update a monitor              | UptimeRobot Tool              | Update monitor details         | UptimeRobot Tool MCP Server |                             |                                     |
| Sticky Note 4                 | Sticky Note                   | Documentation placeholder      |                             |                             |                                     |
| Create a public status page    | UptimeRobot Tool              | Create public status page      | UptimeRobot Tool MCP Server |                             |                                     |
| Delete a public status page    | UptimeRobot Tool              | Delete public status page      | UptimeRobot Tool MCP Server |                             |                                     |
| Get a public status page       | UptimeRobot Tool              | Retrieve public status page    | UptimeRobot Tool MCP Server |                             |                                     |
| Get many public status pages   | UptimeRobot Tool              | List all public status pages   | UptimeRobot Tool MCP Server |                             |                                     |
| Sticky Note 5                 | Sticky Note                   | Documentation placeholder      |                             |                             |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Server Trigger Node**  
   - Add node: **UptimeRobot Tool MCP Server** (Type: MCP Trigger).  
   - Configure webhook (auto-generated webhook ID).  
   - This node acts as the main trigger and routes incoming MCP events.

2. **Add Account Operation Node**  
   - Add node: **Get an account** (Type: UptimeRobot Tool).  
   - Set operation to "Get an account".  
   - Connect input from MCP Server trigger node output.  
   - Configure UptimeRobot API credentials (API key).  

3. **Add Alert Contact Operation Nodes**  
   - For each operation: create, delete, get one, get many, update alert contact:  
     - Add node: UptimeRobot Tool node.  
     - Set operation accordingly (Create an alert contact, Delete an alert contact, etc.).  
     - Connect each node input from MCP Server trigger node output.  
     - Configure UptimeRobot API credentials.  
     - Set required parameters per operation (e.g., contact ID, contact details).

4. **Add Maintenance Window Operation Nodes**  
   - Add nodes for create, get one, get many, update, delete maintenance window.  
   - Configure operation types accordingly.  
   - Connect each input from MCP Server trigger node output.  
   - Configure credentials and parameters (monitor ID, start/end time, etc.).

5. **Add Monitor Operation Nodes**  
   - Add nodes for create, delete, get one, get many, reset, and update monitor.  
   - Set operation parameter accordingly.  
   - Connect inputs from MCP Server trigger node output.  
   - Configure credentials, monitor configuration parameters (type, URL, contacts).

6. **Add Public Status Page Operation Nodes**  
   - Add nodes for create, delete, get one, get many public status pages.  
   - Configure operation parameter accordingly.  
   - Connect inputs from MCP Server trigger node output.  
   - Configure parameters for status page title, monitors, visibility, etc.

7. **Add Sticky Note Nodes (Optional)**  
   - Add sticky notes for documentation placeholders if desired.

8. **Credential Setup**  
   - Ensure UptimeRobot API key credentials are created in n8n.  
   - Assign these credentials to all UptimeRobot Tool nodes.

9. **Testing and Validation**  
   - Test each operation individually by triggering the MCP Server webhook with appropriate commands.  
   - Confirm API responses and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow covers all 21 standard UptimeRobot operations available in the n8n UptimeRobot Tool integration.                                              | UptimeRobot API v2 documentation                     |
| MCP Server trigger node allows easy expansion by adding new operations or routing logic per incoming request type or command.                             | n8n MCP Server documentation                         |
| Ensure API keys used have sufficient permissions to manage all resource types (alerts, monitors, maintenance, status pages).                              | UptimeRobot API key management                        |
| Sticky note nodes are placeholders for documentation or instructions and can be customized per user needs.                                                 | n8n Sticky Note node                                  |

---

**Disclaimer:**  
This document is generated solely based on an automated n8n workflow analysis. It contains no illegal, offensive, or protected elements. All data manipulated is legal and public.