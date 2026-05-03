🛠️ seven Tool MCP Server with both available operations

https://n8nworkflows.xyz/workflows/----seven-tool-mcp-server-with-both-available-operations-5088


# 🛠️ seven Tool MCP Server with both available operations

### 1. Workflow Overview

This workflow implements a **seven Tool MCP Server** that exposes two communication operations via a webhook interface. It is designed to serve as a backend for AI agents or external systems to send SMS messages and convert text to voice calls using the seven Tool API integration within n8n. The workflow includes two main logical blocks corresponding to the two operations supported:

- **1.1 MCP Trigger and Input Reception:** Listens for incoming MCP requests on a webhook, handling AI-driven parameter passing and routing.
- **1.2 SMS Sending Operation:** Processes requests to send SMS messages using the seven Tool SMS API node.
- **1.3 Voice Conversion Operation:** Processes requests to convert text to voice calls using the seven Tool voice API node.

The workflow is pre-configured with zero manual parameter setup, leveraging `$fromAI()` expressions to dynamically extract parameters from AI agent inputs. It features native error handling and formatted responses, ready to be activated and connected to AI agents or external MCP clients.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger and Input Reception

- **Overview:**  
  This block exposes a webhook endpoint `/seven-tool-mcp` that acts as a Multi-Channel Platform (MCP) trigger for AI or external agents. It receives requests, interprets the intended operation (SMS or Voice), and routes inputs accordingly.

- **Nodes Involved:**  
  - `seven Tool MCP Server` (MCP Trigger)

- **Node Details:**  
  - **Node:** `seven Tool MCP Server`  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP trigger node for multi-channel processing)  
  - **Configuration:**  
    - Webhook path set to `/seven-tool-mcp`  
    - Supports two operations defined downstream (SMS send, Voice send)  
  - **Expressions:** None directly, but passes AI parameters downstream via `$fromAI()`  
  - **Input connections:** None (trigger node)  
  - **Output connections:** Connects to both SMS and Voice operation nodes via `ai_tool` output  
  - **Version Requirements:** Requires n8n version supporting MCP triggers and the seven Tool integration  
  - **Potential Failures:**  
    - Webhook authentication or connectivity errors  
    - Invalid or missing AI parameters  
  - **Sub-workflow:** None

#### 2.2 SMS Sending Operation

- **Overview:**  
  This block handles requests to send SMS messages. It extracts parameters like recipient number, sender ID, and message text from AI inputs and sends the SMS via seven Tool’s SMS API node.

- **Nodes Involved:**  
  - `Send an SMS` (seven Tool SMS node)  
  - `Sticky Note 1` (labeling node)

- **Node Details:**  
  - **Node:** `Send an SMS`  
  - **Type:** `n8n-nodes-base.sms77Tool` (seven Tool SMS API node)  
  - **Configuration:**  
    - `to`: dynamically filled by expression `$fromAI('To', '', 'string')`  
    - `from`: dynamically filled by `$fromAI('From', '', 'string')`  
    - `message`: dynamically filled by `$fromAI('Message', '', 'string')`  
    - `options`: left empty (default behavior)  
    - Credentials: Must be configured with valid seven Tool API credentials in one node to enable auth for all  
  - **Input connections:** Receives AI parameters from `seven Tool MCP Server` node via `ai_tool` output  
  - **Output connections:** None (terminal node)  
  - **Version Requirements:** Compatible with n8n versions supporting sms77Tool node and expression evaluation  
  - **Potential Failures:**  
    - Missing or invalid phone numbers or message text  
    - Authentication errors with seven Tool API  
    - Rate limiting or API connectivity issues  
    - Expression evaluation errors if AI input lacks required fields  
  - **Sub-workflow:** None

- **Node:** `Sticky Note 1`  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Purpose:** Visual label "## Sms" to mark the SMS operation block  
  - **No inputs or outputs**

#### 2.3 Voice Conversion Operation

- **Overview:**  
  This block handles requests to convert text messages into voice calls. It captures parameters from AI inputs and uses the seven Tool voice API endpoint to initiate voice calls.

- **Nodes Involved:**  
  - `Convert text to voice` (seven Tool voice API node)  
  - `Sticky Note 2` (labeling node)

- **Node Details:**  
  - **Node:** `Convert text to voice`  
  - **Type:** `n8n-nodes-base.sms77Tool` (seven Tool API node)  
  - **Configuration:**  
    - `to`: dynamically set from `$fromAI('To', '', 'string')`  
    - `message`: dynamically set from `$fromAI('Message', '', 'string')`  
    - `resource`: explicitly set to `voice` to switch API operation to voice call  
    - `options`: empty default  
    - Credentials: same as SMS node, must be configured once  
  - **Input connections:** Receives AI parameters from `seven Tool MCP Server` node via `ai_tool` output  
  - **Output connections:** None (terminal node)  
  - **Version Requirements:** Same as SMS node; requires support for voice resource in sms77Tool node  
  - **Potential Failures:**  
    - Missing or invalid phone numbers or message text  
    - Authentication errors  
    - API limits or call failures  
    - Expression evaluation errors from AI input  
  - **Sub-workflow:** None

- **Node:** `Sticky Note 2`  
  - **Type:** `n8n-nodes-base.stickyNote`  
  - **Purpose:** Label "## Voice" to mark the voice operation block  
  - **No inputs or outputs**

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                 | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                           |
|-----------------------|-----------------------------------|--------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0   | stickyNote                        | Workflow description and setup | None                    | None                    | Contains workflow purpose, setup instructions, usage notes, and support links: [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/), [Discord](https://discord.me/cfomodz) |
| seven Tool MCP Server | @n8n/n8n-nodes-langchain.mcpTrigger | MCP webhook trigger, input routing | None                    | Send an SMS, Convert text to voice |                                                                                                                                       |
| Send an SMS           | n8n-nodes-base.sms77Tool          | Send SMS via seven Tool API    | seven Tool MCP Server    | None                    |                                                                                                                                       |
| Sticky Note 1         | stickyNote                        | Label for SMS operation block  | None                    | None                    | ## Sms                                                                                                                                |
| Convert text to voice | n8n-nodes-base.sms77Tool          | Convert text to voice call     | seven Tool MCP Server    | None                    |                                                                                                                                       |
| Sticky Note 2         | stickyNote                        | Label for Voice operation block | None                    | None                    | ## Voice                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note for Workflow Overview**  
   - Node Type: `stickyNote`  
   - Content:  
     ```
     ## 🛠️ seven Tool MCP Server

     ### 📋 Available Operations (2 total)

     **Sms**: send  
     **Voice**: send

     ### ⚙️ Setup Instructions

     1. Import Workflow into n8n  
     2. Add seven Tool credentials in one tool node (others auto-auth)  
     3. Activate workflow  
     4. Copy webhook URL from MCP trigger node  
     5. Connect AI agents to webhook URL  

     ### ✨ Features

     • Zero config, AI-driven params  
     • Native n8n error & response handling  
     • Modify defaults in nodes if needed  

     ### 💬 Need Help?

     [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)  
     [Discord](https://discord.me/cfomodz)
     ```
   - Position it at approximately (-1460, -100)

2. **Create MCP Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Webhook Path: `seven-tool-mcp`  
   - Position: (-860, -100)  
   - No credentials needed here  
   - This node will expose the webhook for MCP requests

3. **Create Sticky Note for SMS Block**  
   - Node Type: `stickyNote`  
   - Content: `## Sms`  
   - Color: optional (color index 4)  
   - Position: (-1000, 120)

4. **Create SMS Sending Node**  
   - Node Type: `n8n-nodes-base.sms77Tool`  
   - Parameters:  
     - `to`: `={{ $fromAI('To', '', 'string') }}`  
     - `from`: `={{ $fromAI('From', '', 'string') }}`  
     - `message`: `={{ $fromAI('Message', '', 'string') }}`  
     - `options`: leave default empty  
   - Credentials: Configure seven Tool credentials here (only once)  
   - Position: (-800, 140)  
   - Connect input from `seven Tool MCP Server` node's `ai_tool` output

5. **Create Sticky Note for Voice Block**  
   - Node Type: `stickyNote`  
   - Content: `## Voice`  
   - Color: optional (color index 5)  
   - Position: (-1000, 360)

6. **Create Voice Conversion Node**  
   - Node Type: `n8n-nodes-base.sms77Tool`  
   - Parameters:  
     - `to`: `={{ $fromAI('To', '', 'string') }}`  
     - `message`: `={{ $fromAI('Message', '', 'string') }}`  
     - `resource`: set to `voice` (to switch to voice call operation)  
     - `options`: leave default empty  
   - Credentials: Use same seven Tool credentials as SMS node; they share auth  
   - Position: (-800, 380)  
   - Connect input from `seven Tool MCP Server` node's `ai_tool` output

7. **Activate the Workflow**  
   - Ensure all nodes are connected correctly  
   - Verify credentials are valid and tested  
   - Enable the workflow to start accepting MCP requests

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Zero-configuration setup with AI agents automatically populating parameters via `$fromAI()` expressions                     | Allows dynamic parameter input without manual editing                                                                                       |
| Native n8n error handling and response formatting                                                                           | Ensures stable and transparent operation under failure conditions                                                                           |
| For detailed MCP integration guidance and customization support, contact on Discord                                          | https://discord.me/cfomodz                                                                                                                  |
| Official n8n documentation for seven Tool MCP node and usage                                                                | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                                |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This content complies fully with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.