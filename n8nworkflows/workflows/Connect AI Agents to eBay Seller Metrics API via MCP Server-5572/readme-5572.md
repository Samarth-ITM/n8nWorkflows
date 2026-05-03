Connect AI Agents to eBay Seller Metrics API via MCP Server

https://n8nworkflows.xyz/workflows/connect-ai-agents-to-ebay-seller-metrics-api-via-mcp-server-5572


# Connect AI Agents to eBay Seller Metrics API via MCP Server

### 1. Workflow Overview

This workflow, titled **"[eBay] Seller Service Metrics API MCP Server"**, serves as a bridge between AI agents and the eBay Seller Service Metrics API through an MCP (Multi-Channel Platform) server architecture. It enables AI agents to request seller performance metrics from eBay and receive structured responses directly.

The workflow includes the following logical blocks:

- **1.1 Setup and Usage Instructions**  
  Provides operational guidance, setup steps, and customization notes for users deploying the workflow.

- **1.2 MCP Server Trigger**  
  Acts as the entry point that listens for incoming AI agent requests routed via a webhook URL, exposing the API as a service.

- **1.3 Customer Service Metric API Call**  
  Retrieves seller customer service performance data based on dynamic parameters provided by the AI agent.

- **1.4 Seller Standards Profile API Calls**  
  Includes two operations: fetching all seller standards profiles and retrieving a specific seller standards profile by program and cycle.

- **1.5 Traffic Report API Call**  
  Retrieves detailed user traffic analytics for seller listings with dynamic filtering and sorting parameters.

These blocks collectively transform eBay Seller Service Metrics API endpoints into AI-agent-friendly tools accessible through MCP.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Usage Instructions

- **Overview:**  
  This block provides comprehensive setup instructions, usage notes, customization tips, and support contact information. It assists users in importing, configuring, and activating the workflow properly.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Documentation for workflow setup and usage  
    - Content: Stepwise instructions on importing the workflow, configuring OAuth2 credentials, activating the workflow, obtaining the MCP webhook URL, and connecting AI agents. Also includes customization guidance and a Discord support link.  
    - Position: [-1380, -240]  
    - No inputs or outputs (informational only)  
    - Version-Specific: None  
    - Edge Cases: None

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Provides an overview of the API, operations available, and workflow logic  
    - Content: Describes the Analytics API’s purpose, key API areas (Customer Service Metrics, Traffic Reports, Seller Standards Profile), and how the workflow exposes these endpoints via MCP trigger.  
    - Position: [-1120, -240]  
    - No inputs or outputs  
    - Version-Specific: None

#### 2.2 MCP Server Trigger

- **Overview:**  
  This node acts as the webhook trigger for the MCP server interface, receiving incoming AI agent requests and routing them to corresponding API calls.

- **Nodes Involved:**  
  - Seller Service Metrics API MCP Server (MCP Trigger)

- **Node Details:**

  - **Seller Service Metrics API MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: MCP webhook endpoint to receive AI agent requests  
    - Configuration:  
      - Path: `-seller-service-metrics-api--mcp` (used as webhook URL suffix)  
    - Inputs: None (trigger node)  
    - Outputs: Routes to all HTTP Request nodes  
    - Version-Specific: Requires n8n version supporting MCP trigger nodes  
    - Edge Cases:  
      - Webhook URL must be correctly copied and accessible  
      - Authentication setup critical for downstream API requests  
    - Sub-Workflow: None

#### 2.3 Customer Service Metric API Call

- **Overview:**  
  Retrieves seller customer service metrics, such as item not as described or item not received data, based on dynamic path and query parameters.

- **Nodes Involved:**  
  - Sticky Note (label: Customer Service Metric)  
  - Retrieve Seller Service Metrics (HTTP Request Tool)

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Section label for Customer Service Metric API call  
    - Content: "## Customer Service Metric"  
    - Position: [-660, -100]

  - **Retrieve Seller Service Metrics**  
    - Type: HTTP Request Tool  
    - Role: Calls eBay API to fetch customer service metrics  
    - Configuration:  
      - URL Template: `https://api.ebay.com{basePath}/customer_service_metric/{{ $fromAI('customer_service_metric_type') }}/{{ $fromAI('evaluation_type') }}`  
      - Query Parameter: `evaluation_marketplace_id` from AI input  
      - Authentication: Generic HTTP Header Auth (OAuth2 recommended)  
      - Dynamic parameters populated via `$fromAI()` expressions to capture AI agent inputs  
    - Inputs: MCP Trigger node  
    - Outputs: Returns raw API response to MCP trigger (and thus AI agent)  
    - Version-Specific: Supports n8n v1.95+ for HTTP Request Tool v4.2 features  
    - Edge Cases:  
      - Invalid or unsupported metric types or evaluation types may cause API errors  
      - Missing or incorrect OAuth2 credentials can result in authorization failures  
      - Marketplace ID must be valid and supported by eBay Analytics API  
      - Timeout or network issues may occur during API calls

#### 2.4 Seller Standards Profile API Calls

- **Overview:**  
  This block provides two API calls: one to fetch all seller standards profiles and another to retrieve a specific profile filtered by region (program) and evaluation cycle.

- **Nodes Involved:**  
  - Sticky Note2 (label: Seller Standards Profile)  
  - Fetch Seller Standards Profiles (HTTP Request Tool)  
  - Get Seller Standards Profile (HTTP Request Tool)

- **Node Details:**

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Section label for Seller Standards Profile API calls  
    - Content: "## Seller Standards Profile"  
    - Position: [-660, 140]

  - **Fetch Seller Standards Profiles**  
    - Type: HTTP Request Tool  
    - Role: Retrieves all standards profiles for the authenticated seller  
    - Configuration:  
      - URL: `https://api.ebay.com{basePath}/seller_standards_profile` (no path parameters)  
      - Authentication: Generic HTTP Header Auth (OAuth2)  
    - Inputs: MCP Trigger node  
    - Outputs: API response forwarded directly  
    - Edge Cases:  
      - Authentication failure  
      - Empty or no profiles available for seller

  - **Get Seller Standards Profile**  
    - Type: HTTP Request Tool  
    - Role: Retrieves a specific seller standards profile filtered by program and cycle  
    - Configuration:  
      - URL Template: `https://api.ebay.com{basePath}/seller_standards_profile/{{ $fromAI('program') }}/{{ $fromAI('cycle') }}`  
      - Dynamic parameters from AI expressions: program (e.g., PROGRAM_US), cycle (CURRENT or PROJECTED)  
      - Authentication: Generic HTTP Header Auth (OAuth2)  
    - Inputs: MCP Trigger node  
    - Outputs: API response forwarded directly  
    - Edge Cases:  
      - Invalid program or cycle parameters cause 400 errors  
      - Authentication issues  
      - No profile found for given parameters

#### 2.5 Traffic Report API Call

- **Overview:**  
  Retrieves listing traffic reports detailing how often seller listings were viewed, clicked, or purchased, with complex filtering, metrics selection, and sorting options.

- **Nodes Involved:**  
  - Sticky Note3 (label: Traffic Report)  
  - Retrieve Listing Traffic Report (HTTP Request Tool)

- **Node Details:**

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Section label for Traffic Report API call  
    - Content: "## Traffic Report"  
    - Position: [-660, 380]

  - **Retrieve Listing Traffic Report**  
    - Type: HTTP Request Tool  
    - Role: Calls eBay API to obtain detailed traffic data for seller listings  
    - Configuration:  
      - URL: `https://api.ebay.com{basePath}/traffic_report`  
      - Query Parameters: dimension, filter, metric, sort — all dynamically populated via `$fromAI()` expressions  
      - Authentication: Generic HTTP Header Auth (OAuth2)  
    - Inputs: MCP Trigger node  
    - Outputs: API response provided to AI agent via MCP  
    - Edge Cases:  
      - Incorrect or malformed filter parameters causing API errors  
      - Query parameter constraints, e.g., date ranges max 90 days, marketplace IDs validity  
      - Authentication failures  
      - Network timeouts or rate limiting

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                        | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                                          |
|-------------------------------|-----------------------------|-------------------------------------|----------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions             | Sticky Note                 | Setup and usage guidance             | None                             | None                          | ### ⚙️ Setup Instructions 1. Import Workflow 2. Configure Authentication 3. Activate Workflow 4. Get MCP URL 5. Connect AI Agent Usage notes and customization tips. Support: https://discord.me/cfomodz, https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ |
| Workflow Overview             | Sticky Note                 | Workflow purpose and API overview   | None                             | None                          | ## 🛠️ Seller Service Metrics API MCP Server - 4 operations overview                                                                                                                                                                                 |
| Seller Service Metrics API MCP Server | MCP Trigger                | Entry point for AI agent requests   | None                             | Retrieve Seller Service Metrics, Fetch Seller Standards Profiles, Get Seller Standards Profile, Retrieve Listing Traffic Report |                                                                                                                                                                                                                                                      |
| Sticky Note                   | Sticky Note                 | Label for Customer Service Metric   | None                             | None                          | ## Customer Service Metric                                                                                                                                                                                                                           |
| Retrieve Seller Service Metrics | HTTP Request Tool           | Fetch customer service metrics      | Seller Service Metrics API MCP Server | None                          | See API docs: https://developer.ebay.com/devzone/rest/api-ref/analytics/types/MarketplaceIdEnum.html                                                                                                                                                   |
| Sticky Note2                  | Sticky Note                 | Label for Seller Standards Profile  | None                             | None                          | ## Seller Standards Profile                                                                                                                                                                                                                          |
| Fetch Seller Standards Profiles | HTTP Request Tool           | Retrieve all seller standards profiles | Seller Service Metrics API MCP Server | None                          |                                                                                                                                                                                                                                                      |
| Get Seller Standards Profile  | HTTP Request Tool           | Retrieve specific seller standards profile | Seller Service Metrics API MCP Server | None                          |                                                                                                                                                                                                                                                      |
| Sticky Note3                  | Sticky Note                 | Label for Traffic Report            | None                             | None                          | ## Traffic Report                                                                                                                                                                                                                                    |
| Retrieve Listing Traffic Report | HTTP Request Tool           | Obtain traffic report data           | Seller Service Metrics API MCP Server | None                          | See API docs: https://developer.ebay.com/devzone/rest/api-ref/analytics/types/FilterField.html, https://developer.ebay.com/devzone/rest/api-ref/analytics/types/SortField.html                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Setup Instructions" Sticky Note**  
   - Add a Sticky Note node  
   - Content: detailed setup instructions as provided in section 2.1  
   - Position: near top-left for visibility

2. **Create "Workflow Overview" Sticky Note**  
   - Add a Sticky Note node  
   - Content: summary of API, operations, and workflow logic per section 2.1  
   - Position: near Setup Instructions node

3. **Add MCP Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: "Seller Service Metrics API MCP Server"  
   - Set webhook path to `-seller-service-metrics-api--mcp`  
   - Position: center-left  
   - No inputs; this is the workflow trigger

4. **Customer Service Metric Block**  
   - Add Sticky Note labeled "Customer Service Metric" near MCP trigger  
   - Add HTTP Request Tool node named "Retrieve Seller Service Metrics"  
   - Configure URL with template:  
     `https://api.ebay.com{basePath}/customer_service_metric/{{ $fromAI('customer_service_metric_type') }}/{{ $fromAI('evaluation_type') }}`  
   - Add Query Parameter:  
     - Name: `evaluation_marketplace_id`  
     - Value: `{{ $fromAI('evaluation_marketplace_id') }}`  
   - Set Authentication to OAuth2 or Generic HTTP Header Auth with proper eBay credentials  
   - Connect MCP Trigger output to this HTTP Request node input

5. **Seller Standards Profile Block**  
   - Add Sticky Note labeled "Seller Standards Profile"  
   - Add HTTP Request Tool node "Fetch Seller Standards Profiles"  
     - URL: `https://api.ebay.com{basePath}/seller_standards_profile`  
     - Authentication: same as above  
     - Connect MCP Trigger output to this node  
   - Add HTTP Request Tool node "Get Seller Standards Profile"  
     - URL Template:  
       `https://api.ebay.com{basePath}/seller_standards_profile/{{ $fromAI('program') }}/{{ $fromAI('cycle') }}`  
     - Authentication: as above  
     - Connect MCP Trigger output to this node

6. **Traffic Report Block**  
   - Add Sticky Note labeled "Traffic Report"  
   - Add HTTP Request Tool node "Retrieve Listing Traffic Report"  
   - Configure URL: `https://api.ebay.com{basePath}/traffic_report`  
   - Add Query Parameters (all with `$fromAI()` dynamic value):  
     - dimension  
     - filter  
     - metric  
     - sort  
   - Authentication: same OAuth2 or generic header auth  
   - Connect MCP Trigger output to this node

7. **Configure Credentials**  
   - Create OAuth2 credentials for eBay API access with required scopes  
   - Assign these credentials to all HTTP Request Tool nodes

8. **Activate Workflow**  
   - Save and activate workflow  
   - Copy the webhook URL from MCP Trigger node (e.g., `https://<your-n8n-instance>/webhook/-seller-service-metrics-api--mcp`)  
   - Use this URL in your AI agent configuration to route requests

9. **Test Each Endpoint via AI Agent**  
   - Verify dynamic parameters are accepted and responses match eBay API format  
   - Adjust parameter defaults or add data transformation nodes as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automations, join the Discord community: https://discord.me/cfomodz              | Support and community help                                                                                                |
| Official n8n documentation for MCP nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Detailed MCP trigger and tool info                                                                                        |
| eBay Analytics API parameter documentation: https://developer.ebay.com/devzone/rest/api-ref/analytics/types/MarketplaceIdEnum.html | Marketplace ID reference for query parameters                                                                            |
| eBay Analytics API FilterField documentation: https://developer.ebay.com/devzone/rest/api-ref/analytics/types/FilterField.html | Detailed filter parameter explanation                                                                                     |
| eBay Analytics API SortField documentation: https://developer.ebay.com/devzone/rest/api-ref/analytics/types/SortField.html | Sorting options for traffic report                                                                                         |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, adhering strictly to applicable content policies and containing no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.