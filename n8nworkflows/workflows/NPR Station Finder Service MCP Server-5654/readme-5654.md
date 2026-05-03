NPR Station Finder Service MCP Server

https://n8nworkflows.xyz/workflows/npr-station-finder-service-mcp-server-5654


# NPR Station Finder Service MCP Server

### 1. Workflow Overview

This workflow, titled **NPR Station Finder Service MCP Server**, is designed to serve as an MCP (Modular Chatbot Protocol) server endpoint that allows AI agents to access NPR member station information via two main API operations: listing stations and retrieving station details. The workflow acts as a bridge between AI agents and the NPR Station Finder Service API, converting AI requests into HTTP calls and returning responses in the original API structure.

**Target Use Cases:**  
- AI agents querying NPR station data by search terms, location, or station ID.  
- Providing a standardized MCP interface for NPR station data retrieval.

**Logical Blocks:**  
- **1.1 Setup and Documentation:** Contains sticky notes with setup instructions and workflow overview information for users and maintainers.  
- **1.2 MCP Trigger Endpoint:** The entry point receiving AI agent requests via MCP protocol.  
- **1.3 NPR Station Finder Operations:** Two HTTP Request Tool nodes implementing the two API operations (List stations, Retrieve station metadata) with AI-populated parameters.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Documentation

**Overview:**  
This block provides essential information and instructions for users to configure, deploy, and customize the workflow, along with a brief overview of its capabilities and usage notes.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Sticky Note (Station Finder title)

**Node Details:**

- **Setup Instructions (Sticky Note)**  
  - Type: Sticky Note  
  - Role: Provides detailed setup steps including importing the workflow, configuring OAuth2 credentials, activating the workflow, obtaining the MCP webhook URL, and connecting AI agents. Also includes usage notes, customization tips, and support contact links.  
  - Key Content: Stepwise instructions, parameter auto-population via `$fromAI()`, links to Discord support and n8n documentation for MCP nodes.  
  - Inputs/Outputs: None (informational)  
  - Edge Cases: None

- **Workflow Overview (Sticky Note)**  
  - Type: Sticky Note  
  - Role: Describes the workflow’s purpose, how it functions, and lists the two available operations.  
  - Key Content: Explains MCP Trigger as server endpoint, HTTP Request nodes calling NPR API, AI expression usage, and operations supported (List, Retrieve).  
  - Inputs/Outputs: None (informational)  
  - Edge Cases: None

- **Sticky Note (Station Finder)**  
  - Type: Sticky Note  
  - Role: Section header for the station finder operations block.  
  - Inputs/Outputs: None  
  - Edge Cases: None

---

#### 1.2 MCP Trigger Endpoint

**Overview:**  
This node serves as the server endpoint for AI agents to send requests conforming to the MCP protocol. It listens for incoming requests on a specific webhook path and routes them to the appropriate operation nodes.

**Nodes Involved:**  
- NPR Station Finder Service MCP Server (MCP Trigger)

**Node Details:**

- **NPR Station Finder Service MCP Server (MCP Trigger)**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Listens for MCP requests at the webhook path `npr-station-finder-service-mcp` and triggers the workflow accordingly.  
  - Configuration: Webhook path set to `npr-station-finder-service-mcp`. No additional authentication configured here (credential handling is done at HTTP Request nodes).  
  - Inputs: Incoming MCP requests  
  - Outputs: Requests routed to connected nodes (Get Stations 1, Get Station 1) based on AI tool operation requested.  
  - Version-specific: Uses MCP trigger node introduced in n8n 1.95+ with Langchain integration.  
  - Edge Cases: Potential webhook connectivity issues, malformed MCP requests, or unsupported operations sent by AI agents.  
  - Sub-workflow: None

---

#### 1.3 NPR Station Finder Operations

**Overview:**  
This block contains two HTTP Request Tool nodes that implement the NPR Station Finder API endpoints to list stations or retrieve station metadata. Parameters are dynamically populated via `$fromAI()` expressions to allow AI agents to specify query and path parameters.

**Nodes Involved:**  
- Get Stations 1 (HTTP Request Tool)  
- Get Station 1 (HTTP Request Tool)

**Node Details:**

- **Get Stations 1 (HTTP Request Tool)**  
  - Type: `n8n-nodes-base.httpRequestTool`  
  - Role: Calls the NPR API endpoint `https://station.api.npr.org/v3/stations` to list stations filtered by optional parameters.  
  - Configuration:  
    - HTTP Method: GET (default)  
    - URL: Set with expression `=https://station.api.npr.org/v3/stations`  
    - Query Parameters:  
      - `q`: Search terms (station name, network, call letters, zipcode), populated by `$fromAI('q', ...)`  
      - `city` and `state`: Location filters, both needed together, populated by respective `$fromAI()` calls  
      - `lat` and `lon`: Geographic coordinates, both needed together, populated by respective `$fromAI()` calls  
    - Authentication: HTTP Header Auth using configured credential (`Test Header Auth Cred`), likely containing the necessary API key or token.  
    - Tool Description: Provides a detailed explanation of parameters and their usage for this API call.  
  - Input: MCP Trigger node via AI tool routing  
  - Output: Response returned to the MCP trigger, preserving original API response structure.  
  - Edge Cases:  
    - Missing or improperly paired parameters (e.g., lat without lon) may cause API errors or empty results.  
    - Authentication failures due to invalid or expired credentials.  
    - Network timeouts or API downtime.  
    - Expression evaluation failures if `$fromAI()` is missing expected keys.  

- **Get Station 1 (HTTP Request Tool)**  
  - Type: `n8n-nodes-base.httpRequestTool`  
  - Role: Calls the NPR API endpoint `https://station.api.npr.org/v3/stations/{stationId}` to retrieve metadata for a specific station.  
  - Configuration:  
    - HTTP Method: GET (default)  
    - URL: Expression `=https://station.api.npr.org/v3/stations/{{ $fromAI('stationId', ...) }}` dynamically inserts the stationId parameter from AI input.  
    - Authentication: Same HTTP Header Auth credential as above.  
    - Tool Description: Explains the required path parameter `stationId`.  
  - Input: MCP Trigger node via AI tool routing  
  - Output: Response passed back to MCP trigger node.  
  - Edge Cases:  
    - Missing or invalid `stationId` parameter causing 404 or 400 errors.  
    - Authentication failures.  
    - Network or API failures.  
    - Expression failures if `stationId` is not provided or invalid.  

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                       | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                                                                        |
|-----------------------------------|-------------------------------|------------------------------------|----------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                | Sticky Note                   | Setup guide and instructions        | None                             | None                          | ⚙️ Setup Instructions: Steps for import, auth config, activation, MCP URL retrieval, AI agent connection, usage notes, customization, support links |
| Workflow Overview                | Sticky Note                   | Workflow purpose and operation list | None                             | None                          | 🛠️ NPR Station Finder Service MCP Server overview, how it works, available operations                                                              |
| Sticky Note (Station Finder)     | Sticky Note                   | Section header for API operations   | None                             | None                          |                                                                                                                                                    |
| NPR Station Finder Service MCP Server | MCP Trigger                  | MCP webhook endpoint for AI requests | None                             | Get Stations 1, Get Station 1 |                                                                                                                                                    |
| Get Stations 1                   | HTTP Request Tool             | List stations operation              | NPR Station Finder Service MCP Server | NPR Station Finder Service MCP Server | Tool to list stations with AI-populated query parameters, returns original API response                                                           |
| Get Station 1                    | HTTP Request Tool             | Retrieve station metadata operation | NPR Station Finder Service MCP Server | NPR Station Finder Service MCP Server | Tool to get station details by stationId parameter from AI, returns original API response                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Setup and Overview**  
   - Add a Sticky Note named **Setup Instructions** with the full setup text including import steps, auth setup, activation, MCP URL retrieval, AI agent connection, usage notes, customization tips, and support links.  
   - Add a Sticky Note named **Workflow Overview** describing the workflow purpose, MCP trigger role, HTTP request usage, AI expression parameters, and available operations.  
   - Add a small Sticky Note named **Station Finder** as a section header for clarity.

2. **Add MCP Trigger Node**  
   - Create a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Name it **NPR Station Finder Service MCP Server**.  
   - Set the webhook path parameter to `npr-station-finder-service-mcp`.  
   - Leave authentication blank here; handle credentials in HTTP nodes.  
   - This node serves as the entry point for AI agent requests.

3. **Create HTTP Request Tool Node for Listing Stations**  
   - Add an `HTTP Request Tool` node.  
   - Name it **Get Stations 1**.  
   - Set the request URL to `https://station.api.npr.org/v3/stations`.  
   - Use GET method (default).  
   - Add query parameters, each set with expression mode:  
     - `q`: `{{$fromAI('q', 'Search terms to search on; can be a station name, network name, call letters, or zipcode', 'string')}}`  
     - `city`: `{{$fromAI('city', 'A city to look for stations from; intended to be paired with `state`', 'string')}}`  
     - `state`: `{{$fromAI('state', 'A state to look for stations from (using the 2-letter abbreviation); intended to be paired with `city`', 'string')}}`  
     - `lat`: `{{$fromAI('lat', 'A latitude value from a geographic coordinate system; only works if paired with `lon`', 'number')}}`  
     - `lon`: `{{$fromAI('lon', 'A longitude value from a geographic coordinate system; only works if paired with `lat`', 'number')}}`  
   - Configure authentication using HTTP Header Auth credentials with the appropriate API key/token (e.g., named "Test Header Auth Cred").  
   - Add a descriptive tool note explaining the parameters and usage.  
   - Connect the MCP Trigger node’s output to this node’s input via the AI tool connection.

4. **Create HTTP Request Tool Node for Retrieving Station Metadata**  
   - Add another `HTTP Request Tool` node.  
   - Name it **Get Station 1**.  
   - Set the URL with expression:  
     `https://station.api.npr.org/v3/stations/{{$fromAI('stationId', 'The numeric ID of a station', 'number')}}`  
   - Use GET method (default).  
   - Use the same HTTP Header Auth credentials as the prior node.  
   - Add a descriptive tool note about the `stationId` path parameter requirement.  
   - Connect the MCP Trigger node’s output to this node’s input via the AI tool connection.

5. **Activate and Test**  
   - Ensure all nodes are saved and workflow is activated.  
   - Copy the MCP webhook URL from the trigger node.  
   - Configure your AI agent to send MCP requests to this URL with appropriate operation and parameters.  
   - Test both operations with valid and invalid parameters to verify error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automations, ping on [Discord](https://discord.me/cfomodz).     | Support and community help                                                                                               |
| Documentation for MCP nodes in n8n and Langchain integration available at [n8n Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) | Official n8n MCP node documentation                                                                                      |
| Parameters are auto-populated by AI using `$fromAI()` expressions, enabling dynamic request building.| Usage note                                                                                                               |
| This workflow preserves the original API response structure, simplifying downstream processing.      | Implementation detail                                                                                                     |

---

**Disclaimer:**  
The provided text and workflow exclusively derive from an automated n8n workflow creation tool. All data and operations comply with applicable content policies and involve only legal and public data.