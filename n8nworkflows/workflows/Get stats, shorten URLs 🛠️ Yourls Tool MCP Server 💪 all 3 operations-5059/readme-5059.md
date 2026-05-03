Get stats, shorten URLs 🛠️ Yourls Tool MCP Server 💪 all 3 operations

https://n8nworkflows.xyz/workflows/get-stats--shorten-urls-----yourls-tool-mcp-server----all-3-operations-5059


# Get stats, shorten URLs 🛠️ Yourls Tool MCP Server 💪 all 3 operations

---

### 1. Workflow Overview

This workflow titled **"Get stats, shorten URLs 🛠️ Yourls Tool MCP Server 💪 all 3 operations"** is designed to handle three primary URL-related operations using the YOURLS (Your Own URL Shortener) tool:

- Shortening URLs  
- Expanding shortened URLs back to their original form  
- Retrieving statistics for a given shortened URL  

The workflow leverages an MCP (Multi-Channel Processing) trigger node to dynamically route incoming requests to one of the three YOURLS operations. These operations are grouped into a logical block that performs the actual URL manipulations. This design supports flexible interaction via webhook or API calls, making it suitable for integrations requiring URL shortening, expansion, or analytics retrieval.

**Logical Blocks:**

- **1.1 Input Trigger:** Receives and interprets incoming requests, routing them accordingly.  
- **1.2 URL Operations:** Contains three YOURLS tool nodes performing shortening, expanding, and statistics retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block listens for incoming requests via an MCP trigger node and determines which URL operation to execute.

- **Nodes Involved:**  
  - Yourls Tool MCP Server

- **Node Details:**

  - **Yourls Tool MCP Server**  
    - **Type:** MCP Trigger node (from Langchain MCP integration)  
    - **Role:** Entry point that receives webhook/API calls and triggers one of the three URL operation nodes based on the request.  
    - **Configuration:**  
      - Webhook ID set to a unique identifier (`4f094d4c-3a44-46f4-9c5a-5975fd8f04a2`), enabling external calls.  
      - No additional parameters configured, indicating default behavior for MCP trigger.  
    - **Expressions/Variables:** None explicitly configured; it routes based on the MCP channel or input payload.  
    - **Input:** External webhook/API calls.  
    - **Output:** Routes output to the "Expand a URL", "Shorten a URL", or "Get stats for a URL" nodes via the "ai_tool" output channel.  
    - **Version:** Version 1, no special version constraints.  
    - **Potential Failure Points:**  
      - Webhook authentication or permission issues (if protected).  
      - Incoming request format errors causing routing failures.  
      - Network or connectivity issues affecting webhook availability.  
    - **Sub-Workflow:** None.

#### 1.2 URL Operations

- **Overview:**  
  This block contains three YOURLS nodes, each performing a distinct URL-related operation triggered by the MCP server node.

- **Nodes Involved:**  
  - Expand a URL  
  - Shorten a URL  
  - Get stats for a URL

- **Node Details:**

  - **Expand a URL**  
    - **Type:** YOURLS Tool node  
    - **Role:** Expands a shortened YOURLS URL back to its original full URL.  
    - **Configuration:** Uses default YOURLS node settings, likely requiring credentials set up in n8n for YOURLS server access.  
    - **Expressions/Variables:** Dynamically receives the URL to expand from the MCP trigger input.  
    - **Input:** Triggered by "Yourls Tool MCP Server" node via "ai_tool" channel.  
    - **Output:** Returns expanded URL data downstream or to the API caller.  
    - **Version:** Version 1.  
    - **Potential Failures:**  
      - Invalid or malformed shortened URL input.  
      - YOURLS server authentication failures.  
      - Network timeouts or server unavailability.  
    - **Sub-Workflow:** None.

  - **Shorten a URL**  
    - **Type:** YOURLS Tool node  
    - **Role:** Shortens a provided long URL into a YOURLS short URL.  
    - **Configuration:** Uses default YOURLS node settings with credentials for access.  
    - **Expressions/Variables:** Receives the original URL to shorten from MCP trigger input.  
    - **Input:** Triggered by "Yourls Tool MCP Server" node via "ai_tool" channel.  
    - **Output:** Returns shortened URL data.  
    - **Version:** Version 1.  
    - **Potential Failures:**  
      - Invalid input URL format.  
      - Quota or rate limits on YOURLS server.  
      - Authentication or connectivity errors.  
    - **Sub-Workflow:** None.

  - **Get stats for a URL**  
    - **Type:** YOURLS Tool node  
    - **Role:** Retrieves statistics (click counts, access data) for a given YOURLS short URL.  
    - **Configuration:** Default YOURLS settings with valid credentials.  
    - **Expressions/Variables:** Takes the short URL as input from the MCP trigger.  
    - **Input:** Triggered by "Yourls Tool MCP Server" node via "ai_tool" channel.  
    - **Output:** Provides URL statistics data.  
    - **Version:** Version 1.  
    - **Potential Failures:**  
      - URL not found or no stats available.  
      - Authentication failure or insufficient permissions.  
      - Network issues or YOURLS server downtime.  
    - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role             | Input Node(s)         | Output Node(s)                       | Sticky Note |
|-----------------------|----------------------------------|----------------------------|-----------------------|------------------------------------|-------------|
| Workflow Overview      | Sticky Note                      | Documentation placeholder  |                       |                                    |             |
| Yourls Tool MCP Server | MCP Trigger (Langchain MCP)      | Input trigger and router   | External webhook/API   | Expand a URL, Shorten a URL, Get stats for a URL |             |
| Expand a URL          | YOURLS Tool                      | Expands short URLs         | Yourls Tool MCP Server |                                    |             |
| Shorten a URL         | YOURLS Tool                      | Shortens long URLs         | Yourls Tool MCP Server |                                    |             |
| Get stats for a URL    | YOURLS Tool                      | Retrieves URL statistics   | Yourls Tool MCP Server |                                    |             |
| Sticky Note           | Sticky Note                      | Documentation placeholder  |                       |                                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add a new node of type **Langchain MCP Trigger** (usually found under Langchain integrations).  
   - Set a unique **Webhook ID** (e.g., generate a UUID).  
   - Leave other parameters default unless specific channel routing is needed.  
   - This node will serve as the entry point to receive all incoming requests.

2. **Create the YOURLS Tool Nodes (3 total)**  
   For each operation, add a YOURLS Tool node:

   - **Expand a URL**  
     - Node Type: YOURLS Tool  
     - Configure to perform the URL expansion operation (usually a specific action within the node).  
     - Ensure credentials for the YOURLS server are set up in n8n (API URL, API signature/token).  
     - Set the input URL dynamically from the MCP trigger node output.

   - **Shorten a URL**  
     - Node Type: YOURLS Tool  
     - Configure for URL shortening action.  
     - Use the same credentials as above.  
     - Input URL comes from MCP trigger.

   - **Get stats for a URL**  
     - Node Type: YOURLS Tool  
     - Configure to fetch statistics for a given short URL.  
     - Credentials as above.  
     - Input short URL from MCP trigger.

3. **Connect the Nodes**  
   - Connect the **Yourls Tool MCP Server** node’s `ai_tool` output to the input of each of the three YOURLS tool nodes.  
   - This setup allows the trigger to route requests to any of the three operations based on request context.

4. **Credentials Setup**  
   - Ensure the YOURLS Tool nodes have valid credentials configured in n8n:  
     - API endpoint URL of the YOURLS server  
     - API signature or token  
     - Proper permissions to shorten, expand, and get stats

5. **Testing and Validation**  
   - Test the workflow by sending requests to the MCP webhook URL with payloads indicating which operation to perform, including URLs to process.  
   - Validate responses for correctness.

6. **(Optional) Add Sticky Notes**  
   - Add sticky notes for documentation or placeholders as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                       |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow demonstrates integration with YOURLS for all common operations: shorten, expand, stats retrieval. | Workflow description and node usage.                                 |
| The MCP trigger node enables flexible multi-operation routing within a single webhook endpoint.  | Useful for API or chatbot integrations requiring dynamic routing.    |
| YOURLS API documentation: https://yourls.org/#API                                            | Official YOURLS API reference for setting up credentials and calls.  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a workflow automation and integration tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.