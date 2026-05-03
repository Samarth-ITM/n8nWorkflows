Perform, Get Scans 🛠️ urlscan.io Tool MCP Server 💪 all 3 operations

https://n8nworkflows.xyz/workflows/perform--get-scans-----urlscan-io-tool-mcp-server----all-3-operations-5065


# Perform, Get Scans 🛠️ urlscan.io Tool MCP Server 💪 all 3 operations

### 1. Workflow Overview

This workflow integrates with the urlscan.io service to perform three primary operations on URLs: "Perform a scan," "Get a scan," and "Get many scans." It is designed to receive and process scan requests via an MCP (Modular Chatbot Protocol) trigger node, which acts as the input interface. The workflow is logically divided into two main blocks:

- **1.1 Input Reception**: Handling incoming requests through the MCP trigger node.
- **1.2 urlscan.io Operations**: Executing the specific urlscan.io API operations — performing a new scan, retrieving a single scan, and retrieving multiple scans — each implemented as separate nodes but all triggered by the MCP trigger node.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block is responsible for receiving incoming requests via a webhook using the MCP trigger node. It waits for external input that will determine which urlscan.io operation to execute.

- **Nodes Involved:**  
  - `urlscan.io Tool MCP Server`

- **Node Details:**

  - **Node Name:** urlscan.io Tool MCP Server  
    - **Type and Technical Role:** MCP Trigger node; acts as a webhook endpoint to receive input messages formatted for modular chatbot protocol interactions.  
    - **Configuration Choices:** Uses a webhook ID to uniquely identify the webhook. No additional parameters configured, implying it relies on the default MCP trigger behavior.  
    - **Key Expressions/Variables:** None explicitly configured; the node outputs the incoming request data to downstream nodes.  
    - **Input Connections:** None (entry node).  
    - **Output Connections:** Connected as input (`ai_tool`) to three urlscan.io Tool nodes (`Get a scan`, `Get many scans`, `Perform a scan`).  
    - **Version-Specific Requirements:** MCP trigger node requires n8n version supporting MCP integration (commonly n8n 0.203.0+).  
    - **Potential Failures:** Webhook authentication or network failures; malformed MCP requests could cause failures in downstream nodes.  
    - **Sub-workflow Reference:** None.

#### Block 1.2: urlscan.io Operations

- **Overview:**  
  This block executes the three urlscan.io API operations based on input from the MCP trigger. Each operation is handled by a dedicated urlscan.io Tool node, allowing for modular processing depending on the requested action.

- **Nodes Involved:**  
  - `Get a scan`  
  - `Get many scans`  
  - `Perform a scan`

- **Node Details:**

  - **Node Name:** Get a scan  
    - **Type and Technical Role:** urlScanIoTool node configured to retrieve details about a single scan by scan ID or URL.  
    - **Configuration Choices:** No explicit parameters shown; likely uses expressions or input data from MCP trigger to specify the scan to retrieve.  
    - **Key Expressions/Variables:** Expected to consume scan identifier from MCP input data.  
    - **Input Connections:** Receives data from `urlscan.io Tool MCP Server` via the `ai_tool` output.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Requires an active urlscan.io API credential configured in n8n.  
    - **Potential Failures:** API authentication error, invalid or missing scan ID, network timeouts, rate limiting by urlscan.io.  
    - **Sub-workflow Reference:** None.

  - **Node Name:** Get many scans  
    - **Type and Technical Role:** urlScanIoTool node configured to retrieve multiple scan results, potentially filtered or paginated.  
    - **Configuration Choices:** No explicit parameters shown; likely uses MCP input for filtering or pagination parameters.  
    - **Key Expressions/Variables:** Expected to consume filters or query parameters from MCP input.  
    - **Input Connections:** Receives data from `urlscan.io Tool MCP Server` via the `ai_tool` output.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Requires urlscan.io API credential; may need to handle pagination.  
    - **Potential Failures:** API errors, rate limits, invalid query parameters.  
    - **Sub-workflow Reference:** None.

  - **Node Name:** Perform a scan  
    - **Type and Technical Role:** urlScanIoTool node configured to submit a new URL to urlscan.io for scanning.  
    - **Configuration Choices:** No explicit parameters shown; expects URL and options from MCP input.  
    - **Key Expressions/Variables:** URL to scan and any scanning options from MCP payload.  
    - **Input Connections:** Receives data from `urlscan.io Tool MCP Server` via the `ai_tool` output.  
    - **Output Connections:** None (terminal node).  
    - **Version-Specific Requirements:** Requires urlscan.io API credential.  
    - **Potential Failures:** Invalid URL, API authentication error, rate limiting, network failures.  
    - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                         | Input Node(s)               | Output Node(s)                          | Sticky Note |
|-------------------------|---------------------------------|---------------------------------------|-----------------------------|----------------------------------------|-------------|
| Workflow Overview       | Sticky Note                     | Documentation placeholder             | None                        | None                                   |             |
| urlscan.io Tool MCP Server | MCP Trigger                    | Receives incoming MCP webhook requests| None                        | Get a scan, Get many scans, Perform a scan |             |
| Get a scan              | urlScanIoTool                   | Retrieves a single scan from urlscan.io| urlscan.io Tool MCP Server  | None                                   |             |
| Get many scans          | urlScanIoTool                   | Retrieves multiple scans from urlscan.io| urlscan.io Tool MCP Server  | None                                   |             |
| Perform a scan          | urlScanIoTool                   | Performs a new URL scan on urlscan.io | urlscan.io Tool MCP Server  | None                                   |             |
| Sticky Note             | Sticky Note                     | No content                           | None                        | None                                   |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Add a node of type **MCP Trigger** (node type: `@n8n/n8n-nodes-langchain.mcpTrigger`).  
   - Configure with a unique webhook ID (auto-generated or custom).  
   - Leave other parameters at defaults. This node will receive incoming requests specifying which urlscan.io operation to execute.

2. **Create urlScanIoTool Nodes for Operations**

   For each of the following operations, add a node of type **urlScanIoTool** (`n8n-nodes-base.urlScanIoTool`):

   - **Get a scan**  
     - Configure to perform the "Get a scan" operation.  
     - Map input parameters (e.g., scan ID) from the MCP trigger node’s output using expressions.  
     - Ensure the node uses valid urlscan.io API credentials (set up prior in n8n credentials).

   - **Get many scans**  
     - Configure to perform the "Get many scans" operation.  
     - Map filtering or pagination parameters from MCP trigger node output.  
     - Use the same urlscan.io credentials.

   - **Perform a scan**  
     - Configure to perform the "Perform a scan" operation.  
     - Map the URL to scan from MCP trigger node output.  
     - Use the same urlscan.io credentials.

3. **Connect Nodes**  
   - From the MCP Trigger node, connect its output (`ai_tool`) to the input of all three urlScanIoTool nodes (`Get a scan`, `Get many scans`, and `Perform a scan`).  
   - No further connections are necessary unless post-processing or output formatting is required.

4. **Credential Setup**  
   - Create and configure urlscan.io API credentials in n8n (API key/token).  
   - Assign these credentials to each urlScanIoTool node to authorize API requests.

5. **Optional Sticky Notes**  
   - Add sticky notes for documentation or comments as needed. For example, a "Workflow Overview" note at the start.

6. **Testing and Validation**  
   - Test each operation by triggering the webhook with appropriate MCP messages specifying the desired operation and parameters.  
   - Validate API responses and handle errors accordingly in potential future workflow enhancements.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                           |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| This workflow is designed to be triggered via MCP protocol, which is suited for chatbot integration. | MCP protocol documentation: https://github.com/microsoft/microsoft-conversational-ai-protocol |
| urlscan.io API documentation provides details on available endpoints and parameters.          | https://urlscan.io/about-api/                                            |
| Ensure urlscan.io API credentials have sufficient permissions and quota to avoid rate limiting.| urlscan.io API key management page                                       |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.