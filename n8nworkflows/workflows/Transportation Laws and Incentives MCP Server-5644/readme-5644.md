Transportation Laws and Incentives MCP Server

https://n8nworkflows.xyz/workflows/transportation-laws-and-incentives-mcp-server-5644


# Transportation Laws and Incentives MCP Server

### 1. Workflow Overview

This workflow, titled **Transportation Laws and Incentives MCP Server**, provides an MCP (Modular Control Panel) compatible server interface for querying a comprehensive dataset of federal and state transportation laws and incentives related to alternative fuels, vehicles, air quality, and fuel efficiency. It exposes 4 distinct API operations through an MCP trigger node, enabling AI agents to dynamically query data from the U.S. Department of Energy’s Alternative Fuels Data Center API.

The workflow consists of the following logical blocks:

- **1.1 Setup and Documentation**  
  Provides setup instructions, usage tips, and workflow overview via sticky notes for users and integrators.

- **1.2 MCP Server Trigger**  
  The MCP trigger node serves as the workflow’s entry point, exposing a webhook that listens for AI agent requests and routes them to the appropriate API operation nodes.

- **1.3 API Request Operations**  
  Four HTTP Request Tool nodes perform different API calls to the transportation laws and incentives service:  
  - List Laws and Incentives  
  - List Law Categories  
  - Get Jurisdiction Contacts  
  - Get Law Details by ID  

Each API call node dynamically populates parameters using `$fromAI()` expressions, allowing real-time customization of requests based on AI agent inputs.

- **1.4 Output Format Annotations**  
  Sticky notes adjacent to API nodes specify output format details for clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation Block

- **Overview:**  
  This block provides users and maintainers with setup instructions, workflow purpose, usage notes, and customization hints using sticky notes.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  

- **Node Details:**  
  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Documents step-by-step instructions to import, activate, and connect the MCP server workflow, including usage notes on parameters and links for support.  
    - Configuration: Contains detailed markdown content including links to Discord and n8n documentation.  
    - Inputs/Outputs: None (informational node).  
    - Edge Cases: None.  

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Provides a high-level description of the workflow, its data source, and technical approach (MCP trigger, HTTP requests, AI expressions).  
    - Configuration: Markdown content describing workflow purpose and architecture.  
    - Inputs/Outputs: None (informational node).  
    - Edge Cases: None.

---

#### 2.2 MCP Server Trigger Block

- **Overview:**  
  This node acts as the main entry point for AI agents to invoke the workflow via a webhook URL. It listens for incoming requests and routes them to the appropriate HTTP Request Tool nodes based on the requested operation.

- **Nodes Involved:**  
  - Transportation Laws and Incentives MCP Server (MCP Trigger)  

- **Node Details:**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: MCP server webhook trigger that exposes an API endpoint named `transportation-laws-and-incentives-mcp`.  
  - Configuration: Path parameter set as `"transportation-laws-and-incentives-mcp"`, no authentication required.  
  - Input: Incoming HTTP requests from AI agents.  
  - Output: Routes requests to connected HTTP Request Tool nodes via ai_tool connection type.  
  - Version Requirements: Requires n8n version supporting MCP trigger nodes (LangChain integration).  
  - Failure Modes: Webhook unavailability, malformed requests, or MCP protocol errors.  
  - Sub-workflow: None.

---

#### 2.3 API Request Operations Block

- **Overview:**  
  This block contains 4 HTTP Request Tool nodes, each encapsulating a distinct API operation related to transportation laws and incentives. Parameters for API calls are dynamically injected from AI agent inputs using `$fromAI()` expressions. Each node returns API responses in the original structure for direct consumption.

- **Nodes Involved:**  
  - List Laws and Incentives  
  - List Law Categories  
  - Get Jurisdiction Contacts  
  - Get Law Details by ID  

- **Node Details:**  

  - **List Laws and Incentives**  
    - Type: HTTP Request Tool  
    - Role: Fetches a filtered list of laws and incentives from the API based on parameters such as jurisdiction, technology type, incentive type, regulation type, user type, and others.  
    - Configuration:  
      - URL dynamically constructed with response format via `$fromAI('output_format', ...)`.  
      - Query parameters include `api_key`, `limit`, `jurisdiction`, `technology`, `incentive_type`, `regulation_type`, `user_type`, `poc`, `recent`, `expired`, `law_type`, `keyword`, and `local`.  
      - All parameters use `$fromAI()` expressions to receive dynamic user input.  
    - Input: Routed from MCP trigger node.  
    - Output: API JSON or other format response forwarded to AI agent.  
    - Failure Modes: API key invalid/expired, network timeout, invalid parameter values, malformed expressions.  
    - Notes: Limits default to 10 if not provided.  

  - **List Law Categories**  
    - Type: HTTP Request Tool  
    - Role: Retrieves categories of laws for a given category type.  
    - Configuration:  
      - URL uses `$fromAI('output_format', ...)` for response format.  
      - Query parameters: `api_key` and optional `type`.  
      - Uses `$fromAI()` to populate parameters dynamically.  
    - Input: Routed from MCP trigger node.  
    - Output: API response with list of law categories.  
    - Failure Modes: Missing or invalid API key, incorrect category type, network issues.  

  - **Get Jurisdiction Contacts**  
    - Type: HTTP Request Tool  
    - Role: Fetches points of contact (POCs) for specified jurisdictions.  
    - Configuration:  
      - URL composes path with `$fromAI('output_format', ...)`.  
      - Query parameters: `api_key` and required `jurisdiction`.  
      - Parameters dynamically set via `$fromAI()`.  
    - Input: Routed from MCP trigger node.  
    - Output: Contact details for jurisdictions.  
    - Failure Modes: Invalid jurisdiction codes, missing API key, network errors.  

  - **Get Law Details by ID**  
    - Type: HTTP Request Tool  
    - Role: Retrieves detailed information about a specific law identified by its ID.  
    - Configuration:  
      - URL path includes law `id` and `output_format` from `$fromAI()`.  
      - Query parameters: `api_key`, optional `poc` (include points of contact), and `expired` (include expired laws).  
      - All parameters dynamically populated via expressions.  
    - Input: Routed from MCP trigger node.  
    - Output: Detailed law data in requested format.  
    - Failure Modes: Invalid law ID, missing API key, incorrect parameter types, API or network failures.

---

#### 2.4 Output Format Annotations Block

- **Overview:**  
  Sticky notes placed near each API Request node label the expected output format and serve as visual cues for developers.

- **Nodes Involved:**  
  - Grid Note 1 (near List Laws and Incentives)  
  - Grid Note 2 (near List Law Categories)  
  - Grid Note 3 (near Get Jurisdiction Contacts)  
  - Grid Note 4 (near Get Law Details by ID)  

- **Node Details:**  
  - Type: Sticky Note  
  - Role: Display "## Output Format" as a reminder about the response structure of corresponding API calls.  
  - Inputs/Outputs: None.

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                           | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                       |
|----------------------------------|--------------------------------|-----------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------|
| Setup Instructions               | Sticky Note                   | Setup and usage instructions            | None                             | None                            | Contains detailed setup instructions, usage notes, customization tips, and support links          |
| Workflow Overview               | Sticky Note                   | Workflow purpose and high-level overview| None                             | None                            | Describes workflow functionality and architecture                                                |
| Transportation Laws and Incentives MCP Server | MCP Trigger                   | Entry point webhook for AI agent calls | None                             | List Laws and Incentives, List Law Categories, Get Jurisdiction Contacts, Get Law Details by ID |                                                                                                  |
| List Laws and Incentives        | HTTP Request Tool             | Fetch filtered list of laws and incentives | Transportation Laws and Incentives MCP Server | None                            | Full list of laws and incentives matching query parameters                                       |
| List Law Categories             | HTTP Request Tool             | Retrieve law categories by type         | Transportation Laws and Incentives MCP Server | None                            | Law categories for given category type                                                           |
| Get Jurisdiction Contacts       | HTTP Request Tool             | Fetch points of contact for jurisdictions | Transportation Laws and Incentives MCP Server | None                            | Points of contact for specified jurisdictions                                                    |
| Get Law Details by ID           | HTTP Request Tool             | Retrieve detailed information about a law by ID | Transportation Laws and Incentives MCP Server | None                            | Detailed data for law identified by ID                                                          |
| Grid Note 1                    | Sticky Note                   | Output format annotation for List Laws and Incentives | None                             | None                            | ## Output Format                                                                                |
| Grid Note 2                    | Sticky Note                   | Output format annotation for List Law Categories | None                             | None                            | ## Output Format                                                                                |
| Grid Note 3                    | Sticky Note                   | Output format annotation for Get Jurisdiction Contacts | None                             | None                            | ## Output Format                                                                                |
| Grid Note 4                    | Sticky Note                   | Output format annotation for Get Law Details by ID | None                             | None                            | ## Output Format                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Setup and Overview**  
   - Create a Sticky Note node named **Setup Instructions** with the detailed markdown content describing workflow import, activation, MCP URL retrieval, AI agent connection, usage notes, customization tips, and support links.  
   - Create another Sticky Note node named **Workflow Overview** with markdown explaining the workflow’s purpose, data sources, and technical architecture (MCP trigger, API calls, AI expressions).

2. **Add MCP Trigger Node**  
   - Add an MCP Trigger node named **Transportation Laws and Incentives MCP Server**.  
   - Set **Path** parameter to `"transportation-laws-and-incentives-mcp"`.  
   - No authentication required.  
   - This node exposes the webhook URL for AI agents to call.

3. **Add HTTP Request Tool Nodes for API Operations**  

   - **List Laws and Incentives**  
     - Add HTTP Request Tool node named **List Laws and Incentives**.  
     - Set URL to:  
       `http://developer.nrel.gov/api/transportation-incentives-laws/v1.{{$fromAI('output_format', 'Response format', 'string', 'json')}}`  
     - Enable **Send Query** with parameters:  
       - `api_key`: `{{$fromAI('api_key', 'API Key', 'string', 'DEMO_KEY')}}`  
       - `limit`: `{{$fromAI('limit', 'Limit the number of laws returned', 'number', 10)}}`  
       - `jurisdiction`: `{{$fromAI('jurisdiction', 'Return laws for the given Jurisdiction...', 'string')}}`  
       - `technology`: `{{$fromAI('technology', 'Search by the technology type...', 'string')}}`  
       - `incentive_type`: `{{$fromAI('incentive_type', 'Search by the incentive type...', 'string')}}`  
       - `regulation_type`: `{{$fromAI('regulation_type', 'Search by the regulation type...', 'string')}}`  
       - `user_type`: `{{$fromAI('user_type', 'Search by the user type...', 'string')}}`  
       - `poc`: `{{$fromAI('poc', 'Include points of contacts...', 'boolean', false)}}`  
       - `recent`: `{{$fromAI('recent', 'Return only recently added or updated laws...', 'boolean', false)}}`  
       - `expired`: `{{$fromAI('expired', 'true returns only expired laws...', 'boolean', false)}}`  
       - `law_type`: `{{$fromAI('law_type', 'Search by the law type...', 'string')}}`  
       - `keyword`: `{{$fromAI('keyword', 'Search laws by keyword...', 'string')}}`  
       - `local`: `{{$fromAI('local', 'Show only local examples...', 'boolean', false)}}`  

   - **List Law Categories**  
     - Add HTTP Request Tool node named **List Law Categories**.  
     - Set URL to:  
       `http://developer.nrel.gov/api/transportation-incentives-laws/v1/category-list.{{$fromAI('output_format', 'Response format', 'string', 'json')}}`  
     - Enable **Send Query** with parameters:  
       - `api_key`: `{{$fromAI('api_key', 'API Key', 'string', 'DEMO_KEY')}}`  
       - `type`: `{{$fromAI('type', 'Search by the category type.', 'string')}}`  

   - **Get Jurisdiction Contacts**  
     - Add HTTP Request Tool node named **Get Jurisdiction Contacts**.  
     - Set URL to:  
       `http://developer.nrel.gov/api/transportation-incentives-laws/v1/pocs.{{$fromAI('output_format', 'Response format', 'string', 'json')}}`  
     - Enable **Send Query** with parameters:  
       - `api_key`: `{{$fromAI('api_key', 'API Key', 'string', 'DEMO_KEY')}}`  
       - `jurisdiction`: `{{$fromAI('jurisdiction', 'Return the points of contact for the given Jurisdiction...', 'string')}}`  

   - **Get Law Details by ID**  
     - Add HTTP Request Tool node named **Get Law Details by ID**.  
     - Set URL to:  
       `http://developer.nrel.gov/api/transportation-incentives-laws/v1/{{$fromAI('id', 'The id of the law...', 'number')}}.{{$fromAI('output_format', 'Response format', 'string', 'json')}}`  
     - Enable **Send Query** with parameters:  
       - `api_key`: `{{$fromAI('api_key', 'API Key', 'string', 'DEMO_KEY')}}`  
       - `poc`: `{{$fromAI('poc', 'Include points of contacts...', 'boolean', false)}}`  
       - `expired`: `{{$fromAI('expired', 'true returns a record no matter its status...', 'boolean', false)}}`  

4. **Connect Nodes**  
   - Connect the MCP Trigger node’s `ai_tool` output to each of the 4 HTTP Request Tool nodes’ inputs to allow routing requests.  

5. **Add Sticky Notes for Output Format**  
   - For each HTTP Request Tool node, add a Sticky Note nearby with the content `## Output Format` as a reminder of the response structure.

6. **Activate Workflow**  
   - Ensure no authentication is required on the MCP Trigger.  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP Trigger node to configure AI agents to send requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                            |
|-------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Parameters for API calls are auto-populated using `$fromAI()` expressions that dynamically inject AI agent inputs. | Usage note from Setup Instructions sticky note.                                                                             |
| Four distinct API endpoints are exposed as tools via the MCP server interface.                                     | Workflow Overview sticky note.                                                                                              |
| No authentication is required for the MCP server webhook.                                                        | Setup Instructions sticky note.                                                                                            |
| For integration guidance or custom automation support, contact via Discord: https://discord.me/cfomodz           | Setup Instructions sticky note.                                                                                            |
| Additional technical details and MCP node documentation are available at https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions sticky note.                                                                                            |
| The dataset is sourced from the Alternative Fuels Data Center API, which provides federal and state laws and incentives data. | Workflow Overview sticky note.                                                                                              |

---

**Disclaimer:**  
The text provided is extracted exclusively from a workflow automated with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.