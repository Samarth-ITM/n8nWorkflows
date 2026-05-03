File Hash Verification for AI Agents with hashlookup CIRCL API

https://n8nworkflows.xyz/workflows/file-hash-verification-for-ai-agents-with-hashlookup-circl-api-5554


# File Hash Verification for AI Agents with hashlookup CIRCL API

### 1. Workflow Overview

This workflow implements a **File Hash Verification service** using the **hashlookup CIRCL API**, designed to serve as a **Multi-Channel Prompt (MCP) server** endpoint for AI agents. It exposes 11 distinct API operations from the CIRCL Hash Lookup service, enabling AI agents to query file hash information through a standardized interface.

The workflow is logically divided into the following blocks:

- **1.1 Setup & Documentation:** Provides setup instructions, usage notes, and an overview of the service, facilitating user onboarding and maintenance.
- **1.2 MCP Trigger Endpoint:** Acts as the main webhook entry point that listens for AI agent requests and routes them to appropriate API call nodes.
- **1.3 API Operations:** Contains 11 HTTP Request Tool nodes, each corresponding to a unique CIRCL Hash Lookup API endpoint. These nodes perform the actual hash lookup and related queries using parameters dynamically populated by AI expressions.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation

- **Overview:**  
  This block provides comprehensive setup instructions, workflow purpose, usage notes, and operational details for users deploying or modifying the workflow. It ensures clarity on how to activate and integrate the MCP server with AI agents.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  
  - Sticky Note (Default label)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Documentation for users to set up and use the workflow properly.  
    - Content Highlights:  
      - No authentication needed.  
      - Webhook URL retrieval for MCP server connection.  
      - AI-driven parameter auto-population via `$fromAI()` expressions.  
      - Mention of 11 available API endpoints as tools.  
      - Suggestions for adding error handling, logging, or data transformation.  
      - Support contact with Discord link and n8n documentation URL.  
    - Position: Far left on canvas, large vertical size for readability.  
    - Edge Cases: None (informational only).

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: High-level description of the CIRCL MCP server, available API operations, and workflow function.  
    - Content Highlights:  
      - Explanation of the CIRCL Hash Lookup API and its RESTful nature.  
      - Description of MCP Trigger as server endpoint.  
      - List of 11 supported API operations (e.g., Bulk Search, Lookup MD5/SHA1/SHA256, Session management).  
    - Position: Near Setup Instructions, centered vertically.  
    - Edge Cases: None (informational only).

  - **Sticky Note (Default)**  
    - Type: Sticky Note  
    - Role: Labeling or minor organizational note.  
    - Content: "Default"  
    - Position: Above API operation nodes cluster.

#### 1.2 MCP Trigger Endpoint

- **Overview:**  
  This node acts as the central webhook endpoint, receiving incoming AI agent requests. It automatically parses the request, identifies which API operation is requested, and triggers the corresponding HTTP request node.

- **Nodes Involved:**  
  - hashlookup CIRCL MCP Server (MCP Trigger)

- **Node Details:**

  - **hashlookup CIRCL MCP Server**  
    - Type: MCP Trigger (from n8n-nodes-langchain package)  
    - Role: Webhook listener and dispatcher for AI agent requests.  
    - Configuration:  
      - Webhook path: `hashlookup-circl-mcp`  
      - No authentication configured (open endpoint).  
    - Input: Incoming HTTP request from AI agent with parameters.  
    - Output: Routes to 11 downstream HTTP request nodes via AI tool connections.  
    - Version: Requires n8n version supporting MCP triggers (post v1.95.0 recommended).  
    - Edge Cases:  
      - Invalid or missing parameters from AI agent may cause API call failures.  
      - Network errors or webhook misconfiguration could cause request drops.  
    - Sub-workflow: None.  
    - Notes: This is the single entry point for all API operations.

#### 1.3 API Operations

- **Overview:**  
  Eleven HTTP Request Tool nodes perform distinct API calls to the CIRCL Hash Lookup service. Each node corresponds to a documented API endpoint, with parameters dynamically injected using `$fromAI()` expressions, enabling AI-driven queries.

- **Nodes Involved:**  
  - Bulk Search MD5 Hashes  
  - Bulk Search SHA1 Hashes  
  - List SHA1 Children  
  - Get Database Info  
  - Lookup MD5 Hash  
  - Lookup SHA1 Hash  
  - Lookup SHA256 Hash  
  - List SHA1 Parents  
  - Create Search Session  
  - Get Session Results  
  - Get Top Queries

- **Node Details:**

  For each node:

  - **Type:** HTTP Request Tool  
  - **Role:** Executes HTTP requests to specific CIRCL API endpoints based on AI parameters.  
  - **Configuration:**  
    - HTTP Method: Mostly POST or GET as per API spec.  
    - URL: Contains template expressions calling `$fromAI()` to dynamically insert parameters supplied by AI agents.  
    - Tool Description: Summarizes the API endpoint purpose and required parameters.  
    - Options: Default unless specified.  
  - **Input:** Routed from MCP Trigger via AI tool connection.  
  - **Output:** Returns raw API response back to the MCP node, which sends it to AI agent.  
  - **Version:** Compatible with n8n HTTP Request Tool v4.2+.  
  - **Edge Cases:**  
    - Invalid or missing path parameters can cause 4xx errors.  
    - API service downtime or network issues may cause timeouts or 5xx errors.  
    - Bulk search nodes require proper JSON array structure; malformed input can cause failures.  
  - **Sub-workflow:** None.

  Detailed per node:

  1. **Bulk Search MD5 Hashes**  
     - URL: `https://hashlookup.circl.lu/bulk/md5`  
     - Method: POST  
     - Body: JSON array with key `hashes`  
     - Description: Bulk search multiple MD5 hashes.  

  2. **Bulk Search SHA1 Hashes**  
     - URL: `https://hashlookup.circl.lu/bulk/sha1`  
     - Method: POST  
     - Body: JSON array with key `hashes`  
     - Description: Bulk search multiple SHA1 hashes.  

  3. **List SHA1 Children**  
     - URL: `https://hashlookup.circl.lu/children/{{ sha1 }}/{{ count }}/{{ cursor }}`  
     - Method: GET  
     - Path parameters: sha1 (string), count (number), cursor (string)  
     - Description: List children hashes of a given SHA1 hash with pagination.  

  4. **Get Database Info**  
     - URL: `https://hashlookup.circl.lu/info`  
     - Method: GET  
     - Description: Retrieve general info about the hashlookup database.  

  5. **Lookup MD5 Hash**  
     - URL: `https://hashlookup.circl.lu/lookup/md5/{{ md5 }}`  
     - Method: GET  
     - Path parameter: md5 (string)  
     - Description: Lookup details of a specific MD5 hash.  

  6. **Lookup SHA1 Hash**  
     - URL: `https://hashlookup.circl.lu/lookup/sha1/{{ sha1 }}`  
     - Method: GET  
     - Path parameter: sha1 (string)  
     - Description: Lookup details of a specific SHA1 hash.  

  7. **Lookup SHA256 Hash**  
     - URL: `https://hashlookup.circl.lu/lookup/sha256/{{ sha256 }}`  
     - Method: GET  
     - Path parameter: sha256 (string)  
     - Description: Lookup details of a specific SHA256 hash.  

  8. **List SHA1 Parents**  
     - URL: `https://hashlookup.circl.lu/parents/{{ sha1 }}/{{ count }}/{{ cursor }}`  
     - Method: GET  
     - Path parameters: sha1 (string), count (number), cursor (string)  
     - Description: List parent hashes of a given SHA1 hash with pagination.  

  9. **Create Search Session**  
     - URL: `https://hashlookup.circl.lu/session/create/{{ name }}`  
     - Method: GET  
     - Path parameter: name (string)  
     - Description: Creates a session to keep search context, attaches session key to `hashlookup_session` header.  

  10. **Get Session Results**  
      - URL: `https://hashlookup.circl.lu/session/get/{{ name }}`  
      - Method: GET  
      - Path parameter: name (string)  
      - Description: Returns matching and non-matching hashes for the session.  

  11. **Get Top Queries**  
      - URL: `https://hashlookup.circl.lu/stats/top`  
      - Method: GET  
      - Description: Retrieves top 100 most queried hash values.  

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                                   | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                        |
|------------------------|---------------------------|--------------------------------------------------|------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions     | Sticky Note               | Setup and usage documentation                     | None                         | None                           | ⚙️ Setup Instructions with user onboarding, no auth needed, MCP URL usage, customization tips, and Discord support link.          |
| Workflow Overview      | Sticky Note               | High-level workflow and API operations overview  | None                         | None                           | Description of CIRCL API, MCP trigger role, 11 supported endpoints.                                                                |
| Sticky Note (Default)  | Sticky Note               | Labeling / organizational note                    | None                         | None                           | "Default"                                                                                                                         |
| hashlookup CIRCL MCP Server | MCP Trigger              | Webhook entry point for AI requests               | None                         | All 11 HTTP Request Tool nodes | Central MCP server webhook to route requests to API nodes.                                                                         |
| Bulk Search MD5 Hashes | HTTP Request Tool         | Bulk MD5 hashes lookup                            | hashlookup CIRCL MCP Server   | To MCP Server                  | Bulk search API for MD5 hashes, POST JSON array with 'hashes' key.                                                                |
| Bulk Search SHA1 Hashes| HTTP Request Tool         | Bulk SHA1 hashes lookup                           | hashlookup CIRCL MCP Server   | To MCP Server                  | Bulk search API for SHA1 hashes, POST JSON array with 'hashes' key.                                                               |
| List SHA1 Children     | HTTP Request Tool         | Get children of SHA1 hash with pagination        | hashlookup CIRCL MCP Server   | To MCP Server                  | Path params: sha1, count, cursor; paginated children retrieval.                                                                    |
| Get Database Info      | HTTP Request Tool         | Retrieve hashlookup database info                 | hashlookup CIRCL MCP Server   | To MCP Server                  | Simple GET to retrieve database metadata.                                                                                          |
| Lookup MD5 Hash        | HTTP Request Tool         | Lookup details of MD5 hash                         | hashlookup CIRCL MCP Server   | To MCP Server                  | Path param: md5; single hash lookup.                                                                                               |
| Lookup SHA1 Hash       | HTTP Request Tool         | Lookup details of SHA1 hash                        | hashlookup CIRCL MCP Server   | To MCP Server                  | Path param: sha1; single hash lookup.                                                                                              |
| Lookup SHA256 Hash     | HTTP Request Tool         | Lookup details of SHA256 hash                      | hashlookup CIRCL MCP Server   | To MCP Server                  | Path param: sha256; single hash lookup.                                                                                            |
| List SHA1 Parents      | HTTP Request Tool         | Get parents of SHA1 hash with pagination          | hashlookup CIRCL MCP Server   | To MCP Server                  | Path params: sha1, count, cursor; paginated parents retrieval.                                                                     |
| Create Search Session  | HTTP Request Tool         | Start session for contextual search               | hashlookup CIRCL MCP Server   | To MCP Server                  | Path param: name; session key creation.                                                                                            |
| Get Session Results    | HTTP Request Tool         | Get results from session                           | hashlookup CIRCL MCP Server   | To MCP Server                  | Path param: name; fetch session search results.                                                                                    |
| Get Top Queries        | HTTP Request Tool         | Retrieve top 100 most queried hashes               | hashlookup CIRCL MCP Server   | To MCP Server                  | Simple GET; provides popular query stats.                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note Node:**  
   - Name: `Setup Instructions`  
   - Content: Include setup steps, usage notes, customization tips, Discord link, and n8n docs URL.  
   - Position: Left side, large vertical size.

2. **Create Sticky Note Node:**  
   - Name: `Workflow Overview`  
   - Content: Describe the CIRCL Hash Lookup API, MCP server role, and list 11 operations with short descriptions.  
   - Position: Near Setup Instructions.

3. **Create Sticky Note Node:**  
   - Name: `Sticky Note (Default)`  
   - Content: "Default"  
   - Position: Above API operation nodes.

4. **Create MCP Trigger Node:**  
   - Name: `hashlookup CIRCL MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `hashlookup-circl-mcp`  
   - No authentication required.  
   - Position: Center-left.

5. **Create HTTP Request Tool Nodes for API Operations:**  
   For each of the following nodes, set the HTTP method, URL with expressions using `$fromAI()`, and tool description as specified below.

   a. **Bulk Search MD5 Hashes**  
      - Method: POST  
      - URL: `https://hashlookup.circl.lu/bulk/md5`  
      - Tool Description: Bulk search MD5 hashes with JSON array key `hashes`.

   b. **Bulk Search SHA1 Hashes**  
      - Method: POST  
      - URL: `https://hashlookup.circl.lu/bulk/sha1`  
      - Tool Description: Bulk search SHA1 hashes with JSON array key `hashes`.

   c. **List SHA1 Children**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/children/{{ $fromAI('sha1', 'Sha1', 'string') }}/{{ $fromAI('count', 'Count', 'number') }}/{{ $fromAI('cursor', 'Cursor', 'string') }}`  
      - Tool Description: List children of given SHA1 hash with pagination.

   d. **Get Database Info**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/info`  
      - Tool Description: Get database information.

   e. **Lookup MD5 Hash**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/lookup/md5/{{ $fromAI('md5', 'Md5', 'string') }}`  
      - Tool Description: Lookup MD5 hash details.

   f. **Lookup SHA1 Hash**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/lookup/sha1/{{ $fromAI('sha1', 'Sha1', 'string') }}`  
      - Tool Description: Lookup SHA1 hash details.

   g. **Lookup SHA256 Hash**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/lookup/sha256/{{ $fromAI('sha256', 'Sha256', 'string') }}`  
      - Tool Description: Lookup SHA256 hash details.

   h. **List SHA1 Parents**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/parents/{{ $fromAI('sha1', 'Sha1', 'string') }}/{{ $fromAI('count', 'Count', 'number') }}/{{ $fromAI('cursor', 'Cursor', 'string') }}`  
      - Tool Description: List parents of given SHA1 hash with pagination.

   i. **Create Search Session**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/session/create/{{ $fromAI('name', 'Name', 'string') }}`  
      - Tool Description: Create session key attached to session name.

   j. **Get Session Results**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/session/get/{{ $fromAI('name', 'Name', 'string') }}`  
      - Tool Description: Retrieve matching and non-matching hashes from session.

   k. **Get Top Queries**  
      - Method: GET  
      - URL: `https://hashlookup.circl.lu/stats/top`  
      - Tool Description: Get top 100 most queried hashes.

6. **Connect Nodes:**  
   - Connect the MCP Trigger node output (ai_tool type) *to each* of the 11 HTTP Request Tool nodes as input.  
   - No other input nodes for API calls; they all trigger directly from the MCP trigger.

7. **Credentials:**  
   - No authentication or credentials needed for the CIRCL API or the MCP trigger.

8. **Activate Workflow:**  
   - Enable the workflow to start listening on the webhook path `/hashlookup-circl-mcp`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Discord support for integration guidance and custom automations.                                                              | https://discord.me/cfomodz                                                                                                |
| Official n8n documentation for MCP Trigger (Multi-Channel Prompt) nodes and LangChain integration.                            | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                              |
| CIRCL Hash Lookup API supports online queries and offline Bloom filters for local checks.                                      | See CIRCL project documentation for more details.                                                                         |
| Parameters in HTTP Request nodes use `$fromAI()` expressions to auto-populate from AI agent request payloads.                  | This enables seamless dynamic parameter injection without manual editing.                                                 |
| Workflow has no authentication configured; consider adding IP whitelisting or other security measures if exposing publicly.   |                                                                                                                          |

---

**Disclaimer:** The provided information is extracted exclusively from an automated workflow made with n8n, respecting all current content policies. No illegal, offensive, or protected data is included. All data handled is legal and public.