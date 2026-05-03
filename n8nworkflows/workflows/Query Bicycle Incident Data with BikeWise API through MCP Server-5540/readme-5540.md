Query Bicycle Incident Data with BikeWise API through MCP Server

https://n8nworkflows.xyz/workflows/query-bicycle-incident-data-with-bikewise-api-through-mcp-server-5540


# Query Bicycle Incident Data with BikeWise API through MCP Server

### 1. Workflow Overview

This workflow provides an MCP (Multi-Channel Pipeline) server interface to the BikeWise API v2, which exposes bicycling-related incident data. It is designed to be used by AI agents that query bicycling incident and location information through four distinct API endpoints, all integrated into a single MCP-compatible server.

**Target Use Cases:**  
- AI agents requesting bicycling incident data for analysis, reporting, or alerting.  
- Developers requiring a seamless interface to BikeWise API v2 within n8n workflows or automation pipelines.  

**Logical Blocks:**  
- **1.1 Setup and Documentation**: Contains setup instructions and workflow overview notes for users.  
- **1.2 MCP Server Trigger**: The entry point that waits for AI agent requests and routes them to appropriate HTTP API calls.  
- **1.3 Incidents API Operations**: Two HTTP nodes querying the BikeWise API for incident data — paginated incident search and individual incident details.  
- **1.4 Locations API Operations**: Two HTTP nodes returning location data in GeoJSON format with optional styling markers.  

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Documentation

- **Overview:**  
  Provides users with setup instructions, usage notes, customization tips, and a workflow overview describing the purpose and available API operations.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  

- **Node Details:**  
  - **Setup Instructions**  
    - Type: Sticky Note (visual documentation)  
    - Content includes stepwise instructions to import, activate, connect AI agent, and customize.  
    - No input/output connections.  
    - No version-specific requirements.  
    - No failure conditions (informational only).  
  - **Workflow Overview**  
    - Type: Sticky Note  
    - Summarizes the workflow purpose and lists the four operations available via the MCP server.  
    - No input/output connections.  

#### 1.2 MCP Server Trigger

- **Overview:**  
  Acts as the single HTTP webhook endpoint providing the MCP server interface for AI agents to request data. It triggers the workflow execution and routes requests to specific API calls.

- **Nodes Involved:**  
  - BikeWise API v2 MCP Server (MCP Trigger node)  

- **Node Details:**  
  - Type: MCP Trigger (Langchain MCP server trigger)  
  - Configuration: Path set to `bikewise-api-v2-mcp` — this creates a webhook URL that AI agents use as the server endpoint.  
  - Input connections: None (trigger node)  
  - Output connections: Four HTTP request nodes for different BikeWise API endpoints.  
  - Version-specific: Requires n8n with Langchain MCP nodes support.  
  - Potential failure: Webhook misconfiguration, network issues, or MCP runtime errors.  
  - No sub-workflows invoked.  

#### 1.3 Incidents API Operations

- **Overview:**  
  Handles two endpoints related to bicycling incidents: a paginated list of incidents filtered by parameters, and a detailed retrieval of a single incident by ID.

- **Nodes Involved:**  
  - Paginated incidents matching parameters (HTTP Request Tool)  
  - Get incident (HTTP Request Tool)  
  - Sticky Note (“Incidents”)  
  - Sticky Note (“Description - incidents”)  

- **Node Details:**  
  - **Paginated incidents matching parameters**  
    - Type: HTTP Request Tool  
    - Role: Queries `https://bikewise.org/api/v2/incidents` with pagination and filter query parameters.  
    - Query parameters: page, per_page, occurred_before, occurred_after, incident_type, proximity, proximity_square (default 100), query — all dynamically populated via `$fromAI()` expressions, enabling AI-driven parameter injection.  
    - Authentication: Generic HTTP header (no credential needed as API is public).  
    - Output: JSON response with paginated incidents.  
    - Input from MCP Trigger node.  
    - Edge cases: Invalid parameters, API rate limits, network errors.  
  - **Get incident**  
    - Type: HTTP Request Tool  
    - Role: Retrieves detailed information about a specific incident via path parameter `id` dynamically set by AI input.  
    - URL includes dynamic path parameter: `https://bikewise.org/api/v2/incidents/{{id}}`.  
    - Authentication: Generic HTTP header.  
    - Input from MCP Trigger node.  
    - Edge cases: Missing or invalid ID, 404 not found, API downtime.  
  - **Sticky Notes (Incidents, Description - incidents)**  
    - Visual documentation nodes grouping and describing the incident-related nodes.  
    - No inputs/outputs.  

#### 1.4 Locations API Operations

- **Overview:**  
  Provides two endpoints returning locations of bicycling incidents as GeoJSON data: one with raw data and another with simplestyled markers for map visualization.

- **Nodes Involved:**  
  - Unpaginated geojson response (HTTP Request Tool)  
  - Unpaginated geojson response with simplestyled markers (HTTP Request Tool)  
  - Sticky Note (“Locations”)  
  - Sticky Note (“Description - locations”)  

- **Node Details:**  
  - **Unpaginated geojson response**  
    - Type: HTTP Request Tool  
    - Role: Queries `https://bikewise.org/api/v2/locations` with filter query parameters similar to incidents. Supports limit and a boolean `all` flag to ignore limit.  
    - Query parameters: occurred_before, occurred_after, incident_type, proximity, proximity_square (default 100), query, limit (default 100), all (boolean).  
    - Authentication: Generic HTTP header.  
    - Input from MCP Trigger node.  
    - Edge cases: Large result sets, invalid filters, API errors.  
  - **Unpaginated geojson response with simplestyled markers**  
    - Type: HTTP Request Tool  
    - Role: Queries `https://bikewise.org/api/v2/locations/markers` with same parameters as above, returning GeoJSON with additional styling for map markers.  
    - Authentication: Generic HTTP header.  
    - Input from MCP Trigger node.  
    - Edge cases: Same as above.  
  - **Sticky Notes (Locations, Description - locations)**  
    - Visual documentation for location-related endpoints.  

---

### 3. Summary Table

| Node Name                                     | Node Type                  | Functional Role                          | Input Node(s)               | Output Node(s)                                        | Sticky Note                                                                                                                                    |
|-----------------------------------------------|----------------------------|----------------------------------------|-----------------------------|-------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                            | Sticky Note                | Workflow setup and usage instructions  | None                        | None                                                  | ### ⚙️ Setup Instructions: Import workflow, activate, get MCP URL, connect AI agent, usage notes, customization tips, help links.                 |
| Workflow Overview                             | Sticky Note                | Workflow purpose and API operations info| None                        | None                                                  | ## 🛠️ BikeWise API v2 MCP Server ✅ 4 operations: Incident and Location endpoints description.                                                   |
| BikeWise API v2 MCP Server                    | MCP Trigger                | MCP server webhook entry point          | None                        | Paginated incidents matching parameters, Get incident, Unpaginated geojson response, Unpaginated geojson response with simplestyled markers |                                                                                                                                                |
| Sticky Note (Incidents)                       | Sticky Note                | Visual grouping for Incidents block    | None                        | None                                                  | ## Incidents                                                                                                                                    |
| Paginated incidents matching parameters       | HTTP Request Tool          | Fetch paginated list of incidents       | BikeWise API v2 MCP Server  | Returns paginated incident data                        | Parameters auto-populated from AI inputs.                                                                                                     |
| Get incident                                  | HTTP Request Tool          | Fetch detailed incident by ID            | BikeWise API v2 MCP Server  | Returns incident detail JSON                            | Requires valid incident ID parameter.                                                                                                         |
| Description - incidents                       | Sticky Note                | Incidents block description             | None                        | None                                                  | ## 📋 Incidents: Incidents matching parameters                                                                                                 |
| Sticky Note2 (Locations)                      | Sticky Note                | Visual grouping for Locations block    | None                        | None                                                  | ## Locations                                                                                                                                   |
| Unpaginated geojson response                  | HTTP Request Tool          | Fetch unpaginated location data in GeoJSON | BikeWise API v2 MCP Server  | Returns raw GeoJSON location data                      | Supports limit and all flags; parameters auto-populated from AI.                                                                               |
| Unpaginated geojson response with simplestyled markers | HTTP Request Tool          | Fetch location data with styled markers | BikeWise API v2 MCP Server  | Returns GeoJSON with marker styling                    | Same parameters as above; useful for map visualization.                                                                                       |
| Description - locations                       | Sticky Note                | Locations block description             | None                        | None                                                  | ## 📋 Locations: GeoJSON response for matching incidents                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note Node for Setup Instructions**  
   - Type: Sticky Note  
   - Content: Include detailed setup steps, usage notes, customization tips, and help links as provided.  
   - Position: Top-left area for documentation.  

2. **Create Sticky Note Node for Workflow Overview**  
   - Type: Sticky Note  
   - Content: Describe the purpose of the workflow, MCP server functionality, and list available API operations.  
   - Position: Near Setup Instructions.  

3. **Create MCP Trigger Node**  
   - Type: Langchain MCP Trigger  
   - Set `path` parameter to `bikewise-api-v2-mcp` (this will generate a webhook URL).  
   - No credentials needed here.  
   - Place centrally as the main trigger node.  

4. **Create Sticky Note Node for Incidents Group**  
   - Type: Sticky Note  
   - Text: "## Incidents"  
   - Place near the related HTTP request nodes.  

5. **Create HTTP Request Tool Node for Paginated Incidents**  
   - URL: `https://bikewise.org/api/v2/incidents`  
   - Method: GET  
   - Authentication: None (use generic HTTP header auth type with no headers)  
   - Query Parameters:  
     - page: `={{ $fromAI('page', 'Page of results to fetch.', 'number', 1) }}`  
     - per_page: `={{ $fromAI('per_page', 'Number of results to return per page.', 'number') }}`  
     - occurred_before: `={{ $fromAI('occurred_before', 'End of period', 'number') }}`  
     - occurred_after: `={{ $fromAI('occurred_after', 'Start of period', 'number') }}`  
     - incident_type: `={{ $fromAI('incident_type', 'Only incidents of specific type', 'string') }}`  
     - proximity: `={{ $fromAI('proximity', 'Center of location for proximity search', 'string') }}`  
     - proximity_square: `={{ $fromAI('proximity_square', 'Size of the proximity search', 'number', 100) }}`  
     - query: `={{ $fromAI('query', 'Full text search of incidents', 'string') }}`  
   - Connect input from MCP Trigger node.  

6. **Create HTTP Request Tool Node for Single Incident Retrieval**  
   - URL: `https://bikewise.org/api/v2/incidents/{{ $fromAI('id', 'Incident ID', 'number') }}`  
   - Method: GET  
   - Authentication: Same as above.  
   - Connect input from MCP Trigger node.  

7. **Create Sticky Note Node for Incident Description**  
   - Text: "## 📋 Incidents\n\nIncidents matching parameters"  
   - Position near incident HTTP nodes.  

8. **Create Sticky Note Node for Locations Group**  
   - Text: "## Locations"  
   - Position near location HTTP nodes.  

9. **Create HTTP Request Tool Node for Unpaginated GeoJSON Locations**  
   - URL: `https://bikewise.org/api/v2/locations`  
   - Method: GET  
   - Authentication: None (generic HTTP header).  
   - Query Parameters:  
     - occurred_before: `={{ $fromAI('occurred_before', 'End of period', 'number') }}`  
     - occurred_after: `={{ $fromAI('occurred_after', 'Start of period', 'number') }}`  
     - incident_type: `={{ $fromAI('incident_type', 'Only incidents of specific type', 'string') }}`  
     - proximity: `={{ $fromAI('proximity', 'Center of location for proximity search', 'string') }}`  
     - proximity_square: `={{ $fromAI('proximity_square', 'Size of the proximity search', 'number', 100) }}`  
     - query: `={{ $fromAI('query', 'Full text search of incidents', 'string') }}`  
     - limit: `={{ $fromAI('limit', 'Max number of results to return. Defaults to 100', 'number') }}`  
     - all: `={{ $fromAI('all', 'Give âem all to me. Will ignore limit', 'boolean') }}`  
   - Connect input from MCP Trigger node.  

10. **Create HTTP Request Tool Node for Unpaginated GeoJSON Locations with SimpleStyled Markers**  
    - URL: `https://bikewise.org/api/v2/locations/markers`  
    - Method: GET  
    - Authentication: Same as above.  
    - Query Parameters: Same as the previous node.  
    - Connect input from MCP Trigger node.  

11. **Create Sticky Note Node for Locations Description**  
    - Text: "## 📋 Locations\n\nGeoJSON response for matching incidents"  
    - Position near location HTTP nodes.  

12. **Connect all HTTP Request nodes as outputs of the MCP Trigger node**  
    - The MCP Trigger node acts as a dispatcher to all four API endpoint HTTP request nodes.  

13. **Activate the Workflow**  
    - Ensure the MCP Trigger node is active to listen for incoming AI requests.  
    - Copy the webhook URL generated by the MCP Trigger node for AI agent configuration.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Parameters in HTTP nodes are dynamically populated using `$fromAI()` expressions, allowing AI agents to specify API query inputs dynamically.                      | See nodes “Paginated incidents matching parameters” and others.                                                              |
| No authentication required for BikeWise API calls; generic HTTP header auth type is configured but no credentials set.                                              | Public API usage.                                                                                                              |
| Workflow exposes 4 API endpoints as tools to AI agents via a single MCP Trigger webhook URL.                                                                         | Enables seamless AI integration.                                                                                              |
| For troubleshooting or custom automation guidance, connect on Discord: https://discord.me/cfomodz                                                                   | Support channel referenced in Setup Instructions sticky note.                                                                |
| Official n8n documentation on MCP usage with Langchain nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/          | Reference for MCP server setup.                                                                                               |
| Original BikeWise API documentation and source code available on GitHub for deeper API understanding and extensions.                                               | Useful for advanced customizations.                                                                                           |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.