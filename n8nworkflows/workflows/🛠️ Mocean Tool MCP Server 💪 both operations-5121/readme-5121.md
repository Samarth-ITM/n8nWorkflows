🛠️ Mocean Tool MCP Server 💪 both operations

https://n8nworkflows.xyz/workflows/----mocean-tool-mcp-server----both-operations-5121


# 🛠️ Mocean Tool MCP Server 💪 both operations

---
### 1. Workflow Overview

This workflow, titled **🛠️ Mocean Tool MCP Server 💪 both operations**, serves as an MCP (Multi-Channel Platform) server that exposes two primary communication operations: sending SMS and sending Voice messages via the Mocean Tool integration in n8n. It is designed to receive requests through a webhook, process AI-driven parameter inputs, and execute the corresponding messaging operation.

The workflow is logically grouped into the following blocks:

- **1.1 MCP Server Initialization:** Handles incoming webhook requests and routes them for AI-assisted parameter extraction and operation invocation.
- **1.2 SMS Sending Operation:** Processes SMS message sending using parameters dynamically populated by AI.
- **1.3 Voice Message Sending Operation:** Processes Voice message sending similarly, with optional language parameters.

Each operation is exposed via the same MCP webhook, allowing AI agents to call the server with structured requests that automatically map to the correct Mocean Tool action.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Initialization

- **Overview:**  
  This block initializes the MCP server using the `mcpTrigger` node, which listens for incoming requests on a specific webhook path. It acts as the entry point for AI agents to invoke the workflow’s operations.

- **Nodes Involved:**  
  - Mocean Tool MCP Server

- **Node Details:**

  - **Node Name:** Mocean Tool MCP Server  
    - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP trigger node)  
    - **Role:** Listens for HTTP requests on a webhook, processes incoming AI tool calls, and routes to the appropriate child nodes.  
    - **Configuration:**  
      - Webhook path set to `mocean-tool-mcp`.  
      - Automatically handles AI tool invocation and parameter parsing.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Feeds into `Send SMS` and `Send Voice` nodes via ai_tool output.  
    - **Version Requirements:** Requires n8n version supporting `mcpTrigger` node (likely 0.190+).  
    - **Edge Cases / Failure Modes:**  
      - Webhook unavailable if URL changes or workflow inactive.  
      - Improper AI request format may cause parameter extraction failures.  
      - Authentication or credential errors at downstream nodes affect processing.

#### 1.2 SMS Sending Operation

- **Overview:**  
  This block handles sending SMS messages via the Mocean Tool node. It uses AI-extracted parameters for `to`, `from`, and `message` fields to dynamically construct the SMS payload.

- **Nodes Involved:**  
  - Send SMS  
  - Sticky Note 1 (label “Sms”)

- **Node Details:**

  - **Node Name:** Send SMS  
    - **Type:** `n8n-nodes-base.moceanTool`  
    - **Role:** Sends SMS messages using Mocean Tool integration.  
    - **Configuration:**  
      - Parameters `to`, `from`, `message` are dynamically populated from AI input using `$fromAI()` expression with variable names "To", "From", and "Message" respectively.  
      - No additional options configured.  
      - Resource defaults to SMS (implied by absence of `resource` override).  
    - **Input Connections:** Receives AI tool calls from `Mocean Tool MCP Server`.  
    - **Output Connections:** None (end node for SMS flow).  
    - **Credentials:** Requires Mocean Tool credentials configured once; shared across nodes.  
    - **Edge Cases / Failure Modes:**  
      - Missing or invalid phone numbers (`to` or `from`) cause API errors.  
      - Empty or malformed message content may cause failure or no delivery.  
      - Network or API credential issues cause request failures.  
      - Expression failures if AI input variables are absent or malformed.

  - **Node Name:** Sticky Note 1  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Role:** Labels the SMS operation block for clarity in the editor.  
    - **Configuration:** Displays “## Sms” as a header.

#### 1.3 Voice Message Sending Operation

- **Overview:**  
  This block handles sending voice messages through the Mocean Tool node, similarly leveraging AI-extracted parameters. It includes an optional `language` parameter to customize the voice message language.

- **Nodes Involved:**  
  - Send Voice  
  - Sticky Note 2 (label “Voice”)

- **Node Details:**

  - **Node Name:** Send Voice  
    - **Type:** `n8n-nodes-base.moceanTool`  
    - **Role:** Sends voice messages using Mocean Tool integration.  
    - **Configuration:**  
      - Parameters `to`, `from`, `message` dynamically populated from AI input using `$fromAI()` expressions.  
      - `resource` explicitly set to `"voice"` to trigger voice message sending.  
      - `language` parameter left unset (null) by default, can be configured if needed.  
    - **Input Connections:** Receives AI tool calls from `Mocean Tool MCP Server`.  
    - **Output Connections:** None (end node for voice flow).  
    - **Credentials:** Shares Mocean Tool credentials with SMS node.  
    - **Edge Cases / Failure Modes:**  
      - Same as SMS for `to`, `from`, and `message` validation.  
      - Invalid or unsupported language codes can cause errors.  
      - Voice-specific API limits or failures.  
      - Expression failures if AI input variables are missing or malformed.

  - **Node Name:** Sticky Note 2  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Role:** Labels the Voice operation block for clarity.  
    - **Configuration:** Displays “## Voice” as a header.

#### Additional Notes on Sticky Notes

- **Workflow Overview 0:** Provides detailed instructions on importing, setting credentials, activating the workflow, retrieving webhook URL, and connecting AI agents. It also includes useful links to n8n documentation and Discord support for assistance with MCP integration.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                                                            |
|-------------------------|----------------------------------|-------------------------------|------------------------|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0     | Sticky Note                      | Workflow metadata & instructions | None                   | None                   | ## 🛠️ Mocean Tool MCP Server ... [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) ... |
| Mocean Tool MCP Server  | mcpTrigger                       | Entry point webhook for MCP server | None                   | Send SMS, Send Voice   |                                                                                                                                                        |
| Sticky Note 1           | Sticky Note                      | Label for SMS block            | None                   | None                   | ## Sms                                                                                                                                                 |
| Send SMS                | moceanTool                      | Sends SMS messages             | Mocean Tool MCP Server | None                   |                                                                                                                                                        |
| Sticky Note 2           | Sticky Note                      | Label for Voice block          | None                   | None                   | ## Voice                                                                                                                                               |
| Send Voice              | moceanTool                      | Sends Voice messages           | Mocean Tool MCP Server | None                   |                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Sticky Note** node:  
   - Position: top-left  
   - Content: Use the full multi-line text from “Workflow Overview 0” node describing available operations, setup instructions, features, and help links.

3. **Add MCP Trigger Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `Mocean Tool MCP Server`  
   - Parameters:  
     - Set `path` to `mocean-tool-mcp` (this defines webhook URL suffix)  
   - Save to generate webhook URL.  
   - No input connections.

4. **Add Sticky Note for SMS block:**  
   - Text: `## Sms`  
   - Position below MCP Trigger node, left side.

5. **Add Mocean Tool Node for SMS:**  
   - Node Type: `moceanTool`  
   - Name: `Send SMS`  
   - Parameters:  
     - `to`: Expression → `={{ $fromAI('To', ``, 'string') }}`  
     - `from`: Expression → `={{ $fromAI('From', ``, 'string') }}`  
     - `message`: Expression → `={{ $fromAI('Message', ``, 'string') }}`  
     - Leave other options default/empty (default resource is SMS)  
   - Credentials: Configure Mocean Tool API credentials in this node (OAuth or API key as required).  
   - Connect input from `Mocean Tool MCP Server` node’s `ai_tool` output.

6. **Add Sticky Note for Voice block:**  
   - Text: `## Voice`  
   - Position below SMS block, left side.

7. **Add Mocean Tool Node for Voice:**  
   - Node Type: `moceanTool`  
   - Name: `Send Voice`  
   - Parameters:  
     - `to`: Expression → `={{ $fromAI('To', ``, 'string') }}`  
     - `from`: Expression → `={{ $fromAI('From', ``, 'string') }}`  
     - `message`: Expression → `={{ $fromAI('Message', ``, 'string') }}`  
     - `resource`: Set explicitly to `voice` to enable voice message sending  
     - `language`: Leave null or configure as required  
   - Credentials: Use same Mocean Tool credentials as SMS node.  
   - Connect input from `Mocean Tool MCP Server` node’s `ai_tool` output.

8. **Finalize Workflow:**  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP trigger node for use by AI agents.  
   - Ensure Mocean Tool API credentials are valid and authorized for both SMS and Voice operations.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                           |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Workflow includes zero-configuration AI parameter population via `$fromAI()` expressions.          | Core to MCP integration allowing dynamic request parameterization.                                                       |
| For detailed MCP integration support, visit n8n documentation or join the Discord community.       | [n8n MCP Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/), [Discord](https://discord.me/cfomodz) |
| Mocean Tool credentials must be configured once and shared across SMS and Voice nodes.             | Prevents repeated credential setup and simplifies maintenance.                                                            |
| AI agents must call the webhook URL with structured JSON matching MCP specification to trigger operations correctly. | Proper formatting is critical to avoid expression or parameter extraction failures.                                        |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.