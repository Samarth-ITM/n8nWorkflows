🛠️ SIGNL4 Tool MCP Server with both operations

https://n8nworkflows.xyz/workflows/----signl4-tool-mcp-server-with-both-operations-5089


# 🛠️ SIGNL4 Tool MCP Server with both operations

### 1. Workflow Overview

This workflow implements a minimal viable product (MCP) server for the SIGNL4 tool within n8n, enabling automated interaction via two key operations: sending alerts and resolving alerts. It is designed to be triggered via a webhook (MCP trigger node), allowing AI agents or external systems to invoke SIGNL4 alert management functions dynamically.

The workflow is logically organized into the following blocks:

- **1.1 MCP Webhook Trigger:** Listens for incoming requests and provides the entry point for the MCP server.
- **1.2 SIGNL4 Alert Operations:** Implements two separate operations—sending an alert and resolving an alert—each as a dedicated node configured to receive dynamic parameters (mostly from AI agent expressions).
- **1.3 Documentation and Guidance:** Sticky notes provide setup instructions, usage overview, and contextual information for users and maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Webhook Trigger

- **Overview:**  
  This block contains the MCP trigger node set up as a webhook endpoint. It serves as the entry point for the workflow, receiving incoming MCP requests from external sources or AI agents.

- **Nodes Involved:**  
  - SIGNL4 Tool MCP Server (MCP Trigger node)

- **Node Details:**  
  - **Name:** SIGNL4 Tool MCP Server  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP trigger node)  
  - **Configuration:**  
    - Webhook path set to `signl4-tool-mcp`  
    - Listens for incoming MCP calls at this path  
  - **Expressions / Variables:** None (entry node)  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Outputs to "Send an alert" and "Resolve an alert" nodes through AI tool connections  
  - **Version-specific Requirements:** Requires n8n version supporting LangChain MCP trigger nodes  
  - **Potential Failures:**  
    - Webhook URL misconfiguration or access issues  
    - MCP trigger node authentication or permission errors  
  - **Sub-workflow:** None

#### 2.2 SIGNL4 Alert Operations

- **Overview:**  
  This block contains two SIGNL4 Tool nodes that implement the core alert management functionality: sending new alerts and resolving existing alerts. Both nodes receive parameter values dynamically from AI expressions, enabling flexible and automated usage.

- **Nodes Involved:**  
  - Send an alert  
  - Resolve an alert

- **Node Details:**  

  **Send an alert**  
  - **Type:** `n8n-nodes-base.signl4Tool`  
  - **Role:** Sends a new alert to SIGNL4  
  - **Configuration:**  
    - Operation defaults to "send" (implicit)  
    - Message parameter populated dynamically via expression: `$fromAI('Message', '', 'string')` which fetches the 'Message' parameter from AI agent input  
    - No additional fields configured (empty object)  
  - **Inputs:** Receives input from MCP trigger via AI tool connection  
  - **Outputs:** None (end node)  
  - **Potential Failures:**  
    - Authentication errors if SIGNL4 credentials are missing or invalid  
    - API call failures due to network or SIGNL4 service issues  
    - Expression resolution failures if AI input lacks 'Message' field  

  **Resolve an alert**  
  - **Type:** `n8n-nodes-base.signl4Tool`  
  - **Role:** Resolves an existing alert in SIGNL4  
  - **Configuration:**  
    - Operation explicitly set to "resolve"  
    - `externalId` parameter dynamically populated via expression: `$fromAI('External_Id', '', 'string')` to identify the alert to resolve  
  - **Inputs:** Receives input from MCP trigger via AI tool connection  
  - **Outputs:** None (end node)  
  - **Potential Failures:**  
    - Authentication errors if SIGNL4 credentials are missing or invalid  
    - API call failures due to network or SIGNL4 service issues  
    - Expression resolution failures if AI input lacks 'External_Id' field  
    - Resolving non-existent or already resolved alerts may produce errors  

#### 2.3 Documentation and Guidance

- **Overview:**  
  Contains sticky notes that provide users and maintainers with setup instructions, workflow overview, and operation details.

- **Nodes Involved:**  
  - Workflow Overview 0 (large sticky note with full instructions)  
  - Sticky Note 1 (small sticky note labeled "Alert")

- **Node Details:**  

  **Workflow Overview 0**  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Content:**  
    - Title and description of the workflow  
    - Available operations: send and resolve alerts  
    - Stepwise setup instructions including credential configuration and webhook URL usage  
    - Notes on ready-to-use features and AI integration  
    - Help links to n8n documentation and Discord support  
  - **Position:** Top-left for visibility  

  **Sticky Note 1**  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Content:** "Alert" (likely labeling the related nodes visually)  
  - **Position:** Near alert operation nodes  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                      | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                                            |
|-----------------------|----------------------------------|------------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0   | Sticky Note                      | Documentation and Setup Instructions| None                  | None                  | ## 🛠️ SIGNL4 Tool MCP Server<br>### 📋 Available Operations (2 total)<br>**Alert**: send, resolve<br>### ⚙️ Setup Instructions<br>1. Import Workflow<br>2. Add Credentials<br>3. Activate<br>4. Get URL<br>5. Connect AI<br>### ✨ Features<br>• Zero config<br>• AI parameterization<br>• Full operations<br>• Native error handling<br>• Modify defaults<br>### 💬 Help<br>[n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)<br>[Discord](https://discord.me/cfomodz) |
| SIGNL4 Tool MCP Server | MCP Trigger                     | Entry point webhook for MCP server | None                  | Send an alert, Resolve an alert|                                                                                                                        |
| Send an alert         | SIGNL4 Tool                     | Sends new SIGNL4 alert             | SIGNL4 Tool MCP Server | None                  |                                                                                                                        |
| Resolve an alert      | SIGNL4 Tool                     | Resolves existing SIGNL4 alert     | SIGNL4 Tool MCP Server | None                  |                                                                                                                        |
| Sticky Note 1         | Sticky Note                      | Visual label for alert nodes       | None                  | None                  | Alert                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Sticky Note node for Workflow Overview:**  
   - Set width to approximately 420, height to 760.  
   - Paste the detailed setup instructions content as shown in section 3.  
   - Position at top-left for user guidance.

3. **Add MCP Trigger node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name it "SIGNL4 Tool MCP Server".  
   - Set the webhook path to `signl4-tool-mcp`.  
   - No authentication configured here; ensure network accessibility.  
   - Position roughly center-left.

4. **Add SIGNL4 Tool node for sending alerts:**  
   - Node Type: `n8n-nodes-base.signl4Tool`  
   - Name it "Send an alert".  
   - Operation: default is "send" (no need to specify explicitly).  
   - In the "Message" parameter, set the value using expression:  
     `{{$fromAI('Message', '', 'string')}}`  
   - Leave "Additional Fields" empty.  
   - Configure SIGNL4 credentials (see step 6).  
   - Position below and slightly left of MCP trigger node.

5. **Add SIGNL4 Tool node for resolving alerts:**  
   - Node Type: `n8n-nodes-base.signl4Tool`  
   - Name it "Resolve an alert".  
   - Set operation to "resolve".  
   - For the "externalId" parameter, use expression:  
     `{{$fromAI('External_Id', '', 'string')}}`  
   - Configure SIGNL4 credentials (see step 6).  
   - Position below and slightly right of MCP trigger node.

6. **Configure SIGNL4 credentials:**  
   - Create or import SIGNL4 API credentials in n8n.  
   - Assign these credentials to both SIGNL4 Tool nodes.  
   - To ensure credential propagation, configure credentials in one SIGNL4 Tool node first, then open and close others to load credentials.

7. **Connect nodes:**  
   - Connect the MCP Trigger node’s AI tool output to both "Send an alert" and "Resolve an alert" nodes’ AI tool inputs.  
   - This allows the MCP server to route requests dynamically depending on the operation requested.

8. **Add Sticky Note node for alert label:**  
   - Create a small sticky note with content "Alert".  
   - Position near the two SIGNL4 Tool alert nodes for clarity.

9. **Activate the workflow:**  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP trigger node for external use.

10. **Usage:**  
    - Use the MCP webhook URL in your AI agent or external system.  
    - Provide parameters "Message" (string) to send alerts or "External_Id" (string) to resolve alerts through the AI interface.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow supports zero configuration usage with all SIGNL4 alert operations pre-built for immediate use.                    | Workflow Overview sticky note                                                                           |
| AI agents populate parameters dynamically using `$fromAI()` expressions, enabling flexible integration.                         | Workflow Overview sticky note                                                                           |
| For more details on MCP integration, consult the official n8n documentation on MCP nodes.                                       | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/           |
| For community support or customizations, join the Discord server at https://discord.me/cfomodz                                   | Workflow Overview sticky note                                                                           |
| Ensure SIGNL4 credentials are configured correctly in n8n for all nodes using the SIGNL4 Tool node to avoid authentication errors.| General best practice                                                                                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated n8n workflow. All content complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.