EPA Environmental Compliance Data API for AI Agents with MCP Server

https://n8nworkflows.xyz/workflows/epa-environmental-compliance-data-api-for-ai-agents-with-mcp-server-5636


# EPA Environmental Compliance Data API for AI Agents with MCP Server

### 1. Workflow Overview

This workflow serves as a dedicated API backend for AI agents to retrieve and process environmental compliance data from the U.S. EPA Enforcement and Compliance History Online (ECHO) system, specifically focusing on the Resource Conservation and Recovery Act (RCRA) data. It is implemented using n8n’s MCP (Multi-Channel Processing) server node to facilitate AI tool integration and webhooks.

The workflow is logically structured into the following blocks:

- **1.1 MCP Server Trigger**: Entry point that listens for AI agent requests via webhook, initiating the workflow.
- **1.2 RCRA Data Request Blocks**: Multiple paired blocks that handle the request and retrieval of various RCRA datasets from EPA APIs, including:
  - Metadata
  - Facility Search and Details
  - GeoJSON spatial data
  - Data Downloads
  - Map and Info Clusters
  - Paginated Results
- **1.3 Sticky Notes**: Documentation and instruction nodes scattered for setup guidance and workflow overview, enhancing maintainability.

Each data type handled typically involves a two-step process: a "Request" node that issues the HTTP request to the EPA API, followed by a "Get" node to process or retrieve the resultant data. The workflow ensures modularity by separating concerns for different data endpoints.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger Block

- **Overview:**  
  Acts as the workflow’s entry point, receiving incoming requests from AI agents via the MCP server webhook interface.

- **Nodes Involved:**  
  - U.S. EPA Enforcement and Compliance History Online (ECHO) - Resource Conservation and Recovery Act MCP Server

- **Node Details:**  
  - **Type:** MCP Trigger (Langchain n8n node)  
  - **Role:** Listens for and accepts incoming AI agent requests, triggering the workflow.  
  - **Configuration:** Uses a webhook ID for external calls; no additional parameters specified.  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connected downstream to all HTTP Request nodes via ai_tool connections.  
  - **Edge Cases:**  
    - Webhook unavailability or incorrect webhook ID may block workflow activation.  
    - Payload format errors from AI agents could cause downstream failures.  
  - **Sub-Workflow:** None

#### 2.2 RCRA Metadata Retrieval Block

- **Overview:**  
  Retrieves metadata information about the RCRA datasets from the EPA API, supporting AI agents in understanding data context.

- **Nodes Involved:**  
  - Request RCRA Metadata  
  - Get RCRA Metadata

- **Node Details:**  
  - **Request RCRA Metadata**  
    - Type: HTTP Request  
    - Role: Sends HTTP GET request to EPA metadata endpoint (parameters not explicitly defined here, assumed configured for metadata retrieval).  
    - Inputs: Trigger from MCP Server  
    - Outputs: Feeds data to Get RCRA Metadata node  
    - Edge Cases: HTTP errors, API rate limits, malformed responses  
  - **Get RCRA Metadata**  
    - Type: HTTP Request  
    - Role: Processes or fetches the actual metadata content, possibly handling pagination or data parsing after initial request.  
    - Inputs: From Request node  
    - Outputs: Final metadata output for AI consumption  
    - Edge Cases: Data parsing errors, unexpected data formats

#### 2.3 RCRA Facilities Search and Details Block

- **Overview:**  
  Enables searching for EPA RCRA facilities and retrieving detailed information about them.

- **Nodes Involved:**  
  - Request RCRA Facility Search  
  - Search RCRA Facilities  
  - Request RCRA Facility Details  
  - Get RCRA Facility Details

- **Node Details:**  
  - **Request RCRA Facility Search**  
    - Type: HTTP Request  
    - Sends search queries to EPA RCRA facilities API.  
  - **Search RCRA Facilities**  
    - Type: HTTP Request  
    - Retrieves search results following request.  
  - **Request RCRA Facility Details**  
    - Type: HTTP Request  
    - Requests detailed data for specific facilities.  
  - **Get RCRA Facility Details**  
    - Type: HTTP Request  
    - Retrieves and processes detailed data.  
  - **Inputs/Outputs:** Sequenced request-response chain; all nodes receive input from MCP trigger and pass output downstream.  
  - **Edge Cases:**  
    - Query parameter errors, no results found, HTTP failures, data format issues.

#### 2.4 RCRA GeoJSON and Spatial Data Block

- **Overview:**  
  Fetches geospatial data represented in GeoJSON format related to RCRA facilities, enabling spatial analysis.

- **Nodes Involved:**  
  - Request RCRA GeoJSON Data  
  - Get RCRA GeoJSON Data

- **Node Details:**  
  - **Request RCRA GeoJSON Data**  
    - Type: HTTP Request  
    - Requests GeoJSON data from EPA API.  
  - **Get RCRA GeoJSON Data**  
    - Type: HTTP Request  
    - Retrieves and processes GeoJSON payload.  
  - **Edge Cases:** GeoJSON parsing errors, large payload handling, HTTP timeouts.

#### 2.5 RCRA Map and Info Clusters Block

- **Overview:**  
  Retrieves mapping data and clustered information about RCRA facilities for visualization or aggregated analysis.

- **Nodes Involved:**  
  - Request RCRA Map Data  
  - Get RCRA Map Data  
  - Request RCRA Info Clusters  
  - Get RCRA Info Clusters

- **Node Details:**  
  - Each pair follows the request-response pattern.  
  - Map Data nodes handle spatial map datasets; Info Clusters nodes retrieve clustered facility information.  
  - Edge cases include data completeness, API errors, and handling of large datasets.

#### 2.6 RCRA Data Download and Paginated Results Block

- **Overview:**  
  Supports downloading bulk RCRA data files and retrieving paginated query results.

- **Nodes Involved:**  
  - Request RCRA Data Download  
  - Download RCRA Data  
  - Request RCRA Paginated Results  
  - Get RCRA Paginated Results

- **Node Details:**  
  - The data download nodes manage file or dataset downloads from EPA endpoints.  
  - Paginated results nodes handle queries returning multiple pages, managing navigation and data aggregation.  
  - Edge cases: Large file handling, pagination logic errors, download timeouts.

#### 2.7 Sticky Notes

- **Overview:**  
  Provide in-workflow documentation, setup instructions, and overview information to aid users and maintainers.

- **Nodes Involved:**  
  - Setup Instructions (positioned far left)  
  - Workflow Overview (near trigger)  
  - Sticky Note (near trigger)  
  - Sticky Note2 (near metadata nodes)

- **Node Details:**  
  - Type: Sticky Note  
  - Role: Informational only, no data processing  
  - Content: Mostly empty in provided data, likely placeholders for user notes.

---

### 3. Summary Table

| Node Name                                             | Node Type                        | Functional Role                          | Input Node(s)                                               | Output Node(s)                                              | Sticky Note                                  |
|-------------------------------------------------------|---------------------------------|----------------------------------------|-------------------------------------------------------------|-------------------------------------------------------------|----------------------------------------------|
| Setup Instructions                                    | Sticky Note                     | Setup documentation                    | None                                                        | None                                                        |                                              |
| Workflow Overview                                    | Sticky Note                     | Workflow high-level explanation       | None                                                        | None                                                        |                                              |
| U.S. EPA Enforcement and Compliance History Online (ECHO) - Resource Conservation and Recovery Act MCP Server | MCP Trigger                    | Workflow entry point for AI requests  | None                                                        | All HTTP Request nodes via ai_tool connection              |                                              |
| Sticky Note                                           | Sticky Note                     | Informational note                    | None                                                        | None                                                        |                                              |
| Download RCRA Data                                   | HTTP Request                   | Download bulk RCRA data files          | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Data Download                           | HTTP Request                   | Initiate RCRA data download request    | MCP Server trigger                                           | Download RCRA Data                                           |                                              |
| Search RCRA Facilities                               | HTTP Request                   | Retrieve facility search results       | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Facility Search                        | HTTP Request                   | Initiate facility search request       | MCP Server trigger                                           | Search RCRA Facilities                                      |                                              |
| Get RCRA Facility Details                            | HTTP Request                   | Retrieve detailed facility data        | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Facility Details                        | HTTP Request                   | Initiate detailed facility data request | MCP Server trigger                                         | Get RCRA Facility Details                                   |                                              |
| Get RCRA GeoJSON Data                               | HTTP Request                   | Retrieve GeoJSON spatial data          | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA GeoJSON Data                           | HTTP Request                   | Initiate GeoJSON data request          | MCP Server trigger                                           | Get RCRA GeoJSON Data                                      |                                              |
| Get RCRA Info Clusters                              | HTTP Request                   | Retrieve clustered info data           | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Info Clusters                          | HTTP Request                   | Initiate clustered info data request   | MCP Server trigger                                           | Get RCRA Info Clusters                                     |                                              |
| Get RCRA Map Data                                   | HTTP Request                   | Retrieve map data                      | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Map Data                               | HTTP Request                   | Initiate map data request              | MCP Server trigger                                           | Get RCRA Map Data                                          |                                              |
| Get RCRA Paginated Results                          | HTTP Request                   | Retrieve paginated query results       | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Paginated Results                      | HTTP Request                   | Initiate paginated query request       | MCP Server trigger                                           | Get RCRA Paginated Results                                |                                              |
| Sticky Note2                                         | Sticky Note                     | Informational note                    | None                                                        | None                                                        |                                              |
| Get RCRA Metadata                                   | HTTP Request                   | Retrieve metadata info                  | MCP Server trigger                                           | None                                                        |                                              |
| Request RCRA Metadata                               | HTTP Request                   | Initiate metadata info request          | MCP Server trigger                                           | Get RCRA Metadata                                         |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Type: MCP Trigger (Langchain)  
   - Configure webhook ID (auto-generated or custom) to accept AI agent requests.  
   - No additional parameters needed.

2. **Add RCRA Metadata Request Nodes**  
   - Create HTTP Request node named “Request RCRA Metadata”  
     - Method: GET  
     - URL: EPA RCRA metadata endpoint (configure as per EPA API docs)  
   - Create HTTP Request node named “Get RCRA Metadata”  
     - Method: GET or POST depending on API requirements  
     - Configure to process response from “Request RCRA Metadata”  
   - Connect MCP Trigger → Request RCRA Metadata → Get RCRA Metadata

3. **Add RCRA Facility Search and Details Nodes**  
   - Create HTTP Request node “Request RCRA Facility Search” configured for EPA facility search API endpoint.  
   - Create HTTP Request node “Search RCRA Facilities” to retrieve search results.  
   - Create HTTP Request node “Request RCRA Facility Details” for detailed facility info requests.  
   - Create HTTP Request node “Get RCRA Facility Details” for data retrieval and processing.  
   - Connect MCP Trigger → Request RCRA Facility Search → Search RCRA Facilities  
   - Connect MCP Trigger → Request RCRA Facility Details → Get RCRA Facility Details

4. **Add RCRA GeoJSON Data Nodes**  
   - Create HTTP Request node “Request RCRA GeoJSON Data” to request spatial data.  
   - Create HTTP Request node “Get RCRA GeoJSON Data” to obtain and parse GeoJSON.  
   - Connect MCP Trigger → Request RCRA GeoJSON Data → Get RCRA GeoJSON Data

5. **Add RCRA Map and Info Cluster Nodes**  
   - Create “Request RCRA Map Data” and “Get RCRA Map Data” HTTP Request nodes for map datasets.  
   - Create “Request RCRA Info Clusters” and “Get RCRA Info Clusters” HTTP Request nodes for info clusters.  
   - Connect MCP Trigger → Request RCRA Map Data → Get RCRA Map Data  
   - Connect MCP Trigger → Request RCRA Info Clusters → Get RCRA Info Clusters

6. **Add RCRA Data Download and Pagination Nodes**  
   - Create “Request RCRA Data Download” and “Download RCRA Data” HTTP Request nodes for bulk data.  
   - Create “Request RCRA Paginated Results” and “Get RCRA Paginated Results” for paginated API responses.  
   - Connect MCP Trigger → Request RCRA Data Download → Download RCRA Data  
   - Connect MCP Trigger → Request RCRA Paginated Results → Get RCRA Paginated Results

7. **Add Sticky Notes for Documentation**  
   - Add sticky notes titled "Setup Instructions", "Workflow Overview", and additional notes near major blocks.  
   - Use these for setup guidance or workflow description.

8. **Verify all HTTP Request nodes**  
   - Configure URLs, authentication (if required), headers, and parameters according to EPA API documentation for RCRA data.  
   - Test each node independently for successful responses.

9. **Set Workflow Settings**  
   - Timezone: America/New_York  
   - Activate workflow after testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                                      |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses the EPA’s public ECHO API focused on RCRA data endpoints.                                   | EPA Enforcement and Compliance History Online (ECHO): https://echo.epa.gov/                                                        |
| MCP Server node enables AI agent integration via webhooks, facilitating dynamic API calls.                     | n8n MCP Server documentation: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/mcptrigger/                             |
| EPA RCRA datasets include metadata, facility details, spatial GeoJSON data, map data, info clusters, and paginated data. | EPA RCRA API documentation: https://ofmpub.epa.gov/apex/cimc/f?p=100:1::::::                                                        |
| Consider handling API rate limits and response timeouts when calling EPA endpoints in production workflows.    |                                                                                                                                    |

---

**Disclaimer:** The content provided is generated exclusively from an n8n automated workflow analyzing publicly available EPA data. All data usage complies with relevant policies and legal standards.