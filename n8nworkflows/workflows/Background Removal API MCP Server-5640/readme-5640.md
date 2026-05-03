Background Removal API MCP Server

https://n8nworkflows.xyz/workflows/background-removal-api-mcp-server-5640


# Background Removal API MCP Server

### 1. Workflow Overview

This workflow, titled **Background Removal API MCP Server**, serves as an integration interface that exposes the remove.bg background removal API as an MCP (Model Control Protocol) server endpoint. The workflow is designed for AI agents to interact seamlessly with the remove.bg API, enabling three core operations: fetching account balance, removing image backgrounds, and submitting images for improvement.

The logic is divided into these main blocks:

- **1.1 Setup and Documentation**: Provides setup instructions and workflow overview as sticky notes for users.
- **1.2 MCP Trigger Endpoint**: The central entry point receiving AI agent requests via MCP protocol.
- **1.3 API Operations**: Three distinct HTTP request nodes performing:
  - Account balance retrieval
  - Image background removal
  - Image submission for the improvement program

Each operation is connected directly to the MCP trigger node as an AI tool, allowing the AI agent to invoke any of the three endpoints via the MCP server interface.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation Block

- **Overview:** This block provides users with essential setup instructions and a high-level workflow overview directly within n8n using sticky notes. It is purely informational and does not influence workflow execution.
- **Nodes Involved:**
  - Setup Instructions (Sticky Note)
  - Workflow Overview (Sticky Note)
  - Grid Note 1 (Sticky Note)
  - Grid Note 2 (Sticky Note)
  - Grid Note 3 (Sticky Note)

- **Node Details:**

  - **Setup Instructions**
    - Type: Sticky Note
    - Role: Detailed step-by-step instructions on importing, configuring, and activating the workflow, including API key setup and usage notes.
    - Key Content: API key authentication instructions, MCP URL usage, AI expression auto-population, customization tips, and support links.
    - Position: Far left, visually separated for user reference.
    - Edge Cases: None (informational only).

  - **Workflow Overview**
    - Type: Sticky Note
    - Role: Summarizes the workflow purpose, its 3 API operations, and explains the MCP trigger role.
    - Content: Describes the integration of remove.bg API with MCP for AI agents.
    - Edge Cases: None (informational only).

  - **Grid Note 1, 2, 3**
    - Type: Sticky Notes
    - Role: Label and visually group related operation nodes — Fetch Account Info, Improvement Program, Background Removal respectively.
    - Edge Cases: None (informational only).

#### 2.2 MCP Trigger Endpoint Block

- **Overview:** Acts as the single webhook endpoint for AI agents to send MCP requests. It routes incoming MCP tool invocations to the appropriate HTTP request nodes.
- **Nodes Involved:**
  - Background Removal MCP Server (MCP Trigger)

- **Node Details:**

  - **Background Removal MCP Server**
    - Type: MCP Trigger (n8n-nodes-langchain.mcpTrigger)
    - Role: Entry point receiving requests at path `/background-removal-mcp`.
    - Configuration:
      - Webhook path set to "background-removal-mcp".
      - Supports multiple AI tool operations linked downstream.
    - Input: External MCP requests from AI agents.
    - Output: Routes requests to the 3 HTTP request tool nodes based on invoked tool.
    - Edge Cases:
      - Webhook connectivity issues.
      - Authentication failures if API key invalid.
      - Handling unsupported tool names or malformed MCP requests.
    - Version Requirements: Requires n8n version supporting MCP Trigger node (n8n v1.95.3 or later recommended).
  
#### 2.3 API Operations Block

- **Overview:** Contains three HTTP request tool nodes that call specific endpoints of the remove.bg API to perform operations requested by the AI agent.
- **Nodes Involved:**
  - Fetch Account Balance (HTTP Request Tool)
  - Remove Image Background (HTTP Request Tool)
  - Submit Image for Improvement (HTTP Request Tool)

- **Node Details:**

  - **Fetch Account Balance**
    - Type: HTTP Request Tool
    - Role: Fetches user account credit balance and free API calls from remove.bg.
    - Configuration:
      - URL: `https://api.remove.bg/v1.0/account`
      - HTTP Method: GET (default)
      - Authentication: API Key in header (`X-API-Key`)
      - Tool description explains usage.
    - Input: Requests routed from MCP Trigger.
    - Output: Responds with JSON containing account balance.
    - Edge Cases:
      - Invalid API key or expired credentials.
      - Network timeouts.
      - API rate limits.

  - **Remove Image Background**
    - Type: HTTP Request Tool
    - Role: Sends an image to remove.bg to remove its background.
    - Configuration:
      - URL: `https://api.remove.bg/v1.0/removebg`
      - HTTP Method: POST
      - Authentication: API Key in header (`X-API-Key`)
      - Accepts image input parameters auto-populated by AI expressions (`$fromAI()`)
    - Input: Requests routed from MCP Trigger.
    - Output: Returns the processed image or related metadata maintaining original API response structure.
    - Edge Cases:
      - Large image files exceeding API limits.
      - Unsupported image formats.
      - API errors due to malformed requests.
      - Network failures.

  - **Submit Image for Improvement**
    - Type: HTTP Request Tool
    - Role: Submits images to remove.bg Improvement Program for better future background removal.
    - Configuration:
      - URL: `https://api.remove.bg/v1.0/improve`
      - HTTP Method: POST
      - Authentication: API Key in header (`X-API-Key`)
      - Notes emphasize file size limits (12MB), daily submission caps (100 files), and acceptance of program conditions.
    - Input: Requests routed from MCP Trigger.
    - Output: Acknowledgement or response from the improvement API.
    - Edge Cases:
      - Submissions exceeding rate limits.
      - Invalid or missing API keys.
      - File size or format violations.
      - User acceptance of program conditions implicit.
  
---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                       | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                          |
|----------------------------|---------------------------|-------------------------------------|-------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Setup Instructions         | Sticky Note               | Setup and usage instructions         | None                          | None                         | Detailed setup instructions including API key config, MCP URL usage, and support links.                              |
| Workflow Overview          | Sticky Note               | High-level workflow summary          | None                          | None                         | Explains workflow purpose and MCP server integration for background removal API.                                     |
| Grid Note 1                | Sticky Note               | Label for Fetch Account Info         | None                          | None                         | Label grouping for the Fetch Account Balance node.                                                                   |
| Grid Note 2                | Sticky Note               | Label for Improvement Program        | None                          | None                         | Label grouping for the Submit Image for Improvement node.                                                            |
| Grid Note 3                | Sticky Note               | Label for Background Removal         | None                          | None                         | Label grouping for the Remove Image Background node.                                                                  |
| Background Removal MCP Server | MCP Trigger              | MCP webhook entry point               | External AI agent requests     | Fetch Account Balance, Remove Image Background, Submit Image for Improvement |                                                                                                                      |
| Fetch Account Balance      | HTTP Request Tool          | Retrieve account credit balance       | Background Removal MCP Server  | None                         |                                                                                                                      |
| Remove Image Background    | HTTP Request Tool          | Remove background from image          | Background Removal MCP Server  | None                         |                                                                                                                      |
| Submit Image for Improvement | HTTP Request Tool        | Submit image to improvement program   | Background Removal MCP Server  | None                         |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**

   - Add a Sticky Note named **Setup Instructions**:
     - Content: Include detailed setup steps as per original (API key setup, activating workflow, usage notes, customization tips, support links).
   
   - Add a Sticky Note named **Workflow Overview**:
     - Content: Summarize the workflow purpose, MCP trigger role, and the three API operations.
   
   - Add three Sticky Notes named **Grid Note 1**, **Grid Note 2**, **Grid Note 3**:
     - Content: Label the logical groups:
       - Grid Note 1: "Fetch Account Info"
       - Grid Note 2: "Improvement Program"
       - Grid Note 3: "Background Removal"
   
2. **Add MCP Trigger Node:**

   - Node Name: `Background Removal MCP Server`
   - Node Type: `MCP Trigger` (Requires n8n version supporting LangChain MCP nodes)
   - Configuration:
     - Set the webhook path to: `background-removal-mcp`
     - No additional authentication at this node (handled downstream).
   
3. **Add HTTP Request Tool Nodes:**

   - **Fetch Account Balance**
     - Node Type: HTTP Request Tool
     - Name: `Fetch Account Balance`
     - Method: GET (default)
     - URL: `https://api.remove.bg/v1.0/account`
     - Authentication:
       - Use API Key credentials with:
         - Type: API Key in Header
         - Key Name: `X-API-Key`
         - Provide your remove.bg API key
     - Tool Description: "Fetch credit balance and free API calls."
   
   - **Remove Image Background**
     - Node Type: HTTP Request Tool
     - Name: `Remove Image Background`
     - Method: POST
     - URL: `https://api.remove.bg/v1.0/removebg`
     - Authentication:
       - Same API Key credentials as above
     - Parameters:
       - Set to accept image input parameters expected by remove.bg API
       - Use expressions such as `$fromAI()` to allow the AI agent to supply parameters dynamically.
     - Tool Description: "Remove the background of an image."
   
   - **Submit Image for Improvement**
     - Node Type: HTTP Request Tool
     - Name: `Submit Image for Improvement`
     - Method: POST
     - URL: `https://api.remove.bg/v1.0/improve`
     - Authentication:
       - Same API Key credentials as above
     - Parameters:
       - Accept image file input from AI agent via `$fromAI()` expressions.
     - Tool Description: Includes notes on file size (max 12MB), daily limits, and submission conditions.
   
4. **Connect Nodes:**

   - Connect the `Background Removal MCP Server` node’s output as AI tool to each of the three HTTP Request nodes:
     - Fetch Account Balance
     - Remove Image Background
     - Submit Image for Improvement

5. **Credential Setup:**

   - Create an API Key credential in n8n:
     - Type: API Key (header)
     - Header key name: `X-API-Key`
     - Value: Your valid remove.bg API key

6. **Activate Workflow:**

   - Save and activate the workflow.
   - Copy the webhook URL from the MCP Trigger node (should be something like `https://your-n8n-instance/webhook/background-removal-mcp`).
   - Configure your AI agent or client to send MCP requests to this URL to invoke the desired operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup instructions include detailed guidance on API key setup and usage with MCP triggers.                                            | Workflow Setup Sticky Note                                                                       |
| AI expressions like `$fromAI()` are used to automatically populate parameters in HTTP requests from AI agent inputs.                 | Usage Notes in Setup Instructions Sticky Note                                                  |
| MCP Trigger node requires n8n version supporting LangChain MCP nodes, recommended v1.95.3 or later.                                  | Node type requirements                                                                          |
| For integration help or custom automation requests, contact on Discord: https://discord.me/cfomodz                                     | Support Contact                                                                                |
| Official n8n documentation for MCP and LangChain integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Documentation link                                                                             |
| The Improvement Program submission has daily limits (100 files), max file size (12MB), and requires user acceptance of conditions.    | Submit Image for Improvement node description                                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.