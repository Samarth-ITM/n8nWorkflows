🛠️ Emelia Tool MCP Server 💪 all 9 operations

https://n8nworkflows.xyz/workflows/----emelia-tool-mcp-server----all-9-operations-5272


# 🛠️ Emelia Tool MCP Server 💪 all 9 operations

### 1. Workflow Overview

This workflow, titled **"Emelia Tool MCP Server"**, serves as a centralized server handling all nine core operations related to campaigns and contact lists within the Emelia Tool environment. The primary use case is to provide a comprehensive API-like interface that triggers specific campaign or contact list operations upon receiving requests.

The logic is grouped into two main functional blocks:

- **1.1 MCP Trigger Input Reception:**  
  The workflow starts with a specialized trigger node that listens for incoming requests, acting as the entry point for all operations.

- **1.2 Campaigns and Contact Lists Operations:**  
  Based on the trigger input, the workflow routes to one of nine Emelia Tool nodes, each performing a distinct operation such as creating, duplicating, pausing campaigns, or managing contact lists. These nodes handle the core business logic interfacing with Emelia’s API.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger Input Reception

**Overview:**  
This block receives incoming requests to the workflow via a dedicated MCP (Multi Campaign Platform) Trigger node. It acts as the API endpoint that starts the workflow and routes requests to the appropriate operation nodes.

**Nodes Involved:**  
- Emelia Tool MCP Server (MCP Trigger)

**Node Details:**

- **Emelia Tool MCP Server**  
  - **Type:** MCP Trigger (Custom node from @n8n/n8n-nodes-langchain)  
  - **Role:** Workflow entry point that listens for incoming MCP requests (likely HTTP webhook-based).  
  - **Configuration:** The node is configured with an assigned webhook ID to receive external calls. No additional parameters are set, implying reliance on default MCP trigger behavior.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Routes to all subsequent Emelia Tool nodes via "ai_tool" type connections.  
  - **Version Requirements:** Requires n8n version supporting MCP trigger nodes and the @n8n/n8n-nodes-langchain package installed.  
  - **Potential Failures:** Webhook misconfiguration, network issues, invalid request payloads, or authentication errors at the MCP server level.  
  - **Sub-workflows:** None.

---

#### 1.2 Campaigns and Contact Lists Operations

**Overview:**  
This block contains nodes that perform specific operations on campaigns and contact lists within Emelia. Each node is an instance of the Emelia Tool node configured for a single operation. The MCP Trigger node routes inputs to these nodes based on requested operations.

**Nodes Involved:**  
- Add a contact to a campaign  
- Create a campaign  
- Duplicate a campaign  
- Get a campaign  
- Get many campaigns  
- Pause a campaign  
- Start a campaign  
- Add a contact list  
- Get many contact lists

**Node Details:**

- **Add a contact to a campaign**  
  - **Type:** Emelia Tool Node  
  - **Role:** Adds a contact to an existing campaign.  
  - **Configuration:** Uses default parameters; actual contact and campaign details are expected to be provided dynamically via incoming data from the trigger.  
  - **Inputs:** Connected from MCP Trigger via "ai_tool".  
  - **Outputs:** None specified (endpoint operation).  
  - **Failures:** API errors such as invalid campaign ID, contact data issues, or authentication failures.

- **Create a campaign**  
  - **Type:** Emelia Tool Node  
  - **Role:** Creates a new campaign within Emelia.  
  - **Configuration:** Parameters expected dynamically from input.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** Validation errors on campaign parameters, API errors.

- **Duplicate a campaign**  
  - **Type:** Emelia Tool Node  
  - **Role:** Creates a copy of an existing campaign.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** Invalid source campaign ID, API errors.

- **Get a campaign**  
  - **Type:** Emelia Tool Node  
  - **Role:** Retrieves details of a specific campaign.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** Campaign not found, permission issues.

- **Get many campaigns**  
  - **Type:** Emelia Tool Node  
  - **Role:** Retrieves a list of campaigns, possibly paginated or filtered.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** API rate limits, malformed query.

- **Pause a campaign**  
  - **Type:** Emelia Tool Node  
  - **Role:** Pauses an active campaign.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** Campaign already paused or invalid state.

- **Start a campaign**  
  - **Type:** Emelia Tool Node  
  - **Role:** Initiates or resumes a campaign.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** Campaign already active or invalid state.

- **Add a contact list**  
  - **Type:** Emelia Tool Node  
  - **Role:** Adds a new contact list to the system.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** Validation errors on contact list data.

- **Get many contact lists**  
  - **Type:** Emelia Tool Node  
  - **Role:** Retrieves multiple contact lists.  
  - **Inputs:** From MCP Trigger.  
  - **Failures:** API errors, pagination issues.

**Additional Notes:**  
- All these nodes have no complex branching or data transformation within this workflow; they rely on the MCP Trigger to supply correct inputs and handle outputs externally.  
- Two sticky notes are present near the campaign and contact list nodes but contain no content.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                     | Input Node(s)          | Output Node(s)                    | Sticky Note          |
|----------------------------|-----------------------------------|-----------------------------------|-----------------------|----------------------------------|----------------------|
| Workflow Overview 0        | Sticky Note                       | Informational placeholder         | None                  | None                             |                      |
| Emelia Tool MCP Server     | MCP Trigger                      | Entry trigger for all operations  | None                  | All Emelia Tool operation nodes  |                      |
| Add a contact to a campaign| Emelia Tool                      | Add contact to campaign            | Emelia Tool MCP Server | None                             |                      |
| Create a campaign          | Emelia Tool                      | Create a new campaign              | Emelia Tool MCP Server | None                             |                      |
| Duplicate a campaign       | Emelia Tool                      | Duplicate existing campaign        | Emelia Tool MCP Server | None                             |                      |
| Get a campaign             | Emelia Tool                      | Retrieve campaign details          | Emelia Tool MCP Server | None                             |                      |
| Get many campaigns         | Emelia Tool                      | Retrieve multiple campaigns        | Emelia Tool MCP Server | None                             |                      |
| Pause a campaign           | Emelia Tool                      | Pause a campaign                   | Emelia Tool MCP Server | None                             |                      |
| Start a campaign           | Emelia Tool                      | Start or resume a campaign         | Emelia Tool MCP Server | None                             |                      |
| Sticky Note 1              | Sticky Note                       | Informational placeholder         | None                  | None                             |                      |
| Add a contact list         | Emelia Tool                      | Add a new contact list             | Emelia Tool MCP Server | None                             |                      |
| Get many contact lists     | Emelia Tool                      | Retrieve multiple contact lists    | Emelia Tool MCP Server | None                             |                      |
| Sticky Note 2              | Sticky Note                       | Informational placeholder         | None                  | None                             |                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**  
   - Add an **MCP Trigger** node (from @n8n/n8n-nodes-langchain package).  
   - Assign a webhook ID or accept the default generated one.  
   - No custom parameters required.

2. **Add Emelia Tool Operation Nodes:**  
   For each of the nine operations below, add an **Emelia Tool** node with the operation name as the node name. No preset parameters are necessary; inputs will be dynamically received via MCP Trigger:  
   - Add a contact to a campaign  
   - Create a campaign  
   - Duplicate a campaign  
   - Get a campaign  
   - Get many campaigns  
   - Pause a campaign  
   - Start a campaign  
   - Add a contact list  
   - Get many contact lists

3. **Connect Nodes:**  
   - Connect the output of the MCP Trigger node to the input of each Emelia Tool node using the connection type labeled "ai_tool". This implies all operation nodes listen for MCP Trigger inputs and act accordingly.

4. **Add Sticky Notes (Optional):**  
   - Optionally, add sticky notes as placeholders or for future documentation near campaign-related nodes and contact list nodes.

5. **Credentials Setup:**  
   - Configure Emelia Tool credentials in n8n to enable API access. This typically includes API keys or OAuth tokens specific to the Emelia service.  
   - Ensure the MCP Trigger node’s webhook is accessible publicly or within your network as needed.

6. **Test the Workflow:**  
   - Trigger the MCP webhook with payloads specifying which operation to perform and required parameters (e.g., campaign ID, contact details).  
   - Verify each Emelia Tool node performs its intended action and returns expected results.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link           |
|------------------------------------------------------------------------------|---------------------------|
| This workflow utilizes the Emelia Tool nodes and the MCP Trigger node from the @n8n/n8n-nodes-langchain package. | n8n community packages documentation |
| Ensure the external system invoking the MCP Trigger webhook formats requests correctly to route to desired operation nodes. | API request design best practices |
| Sticky notes present have no content but can be used to document each operation’s purpose in a future iteration. | n8n workflow documentation conventions |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.