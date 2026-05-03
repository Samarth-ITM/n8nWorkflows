Connect AI Agents to Canada Holidays API with 6 Province and Holiday Operations

https://n8nworkflows.xyz/workflows/connect-ai-agents-to-canada-holidays-api-with-6-province-and-holiday-operations-5549


# Connect AI Agents to Canada Holidays API with 6 Province and Holiday Operations

### 1. Workflow Overview

This workflow, titled **"Canada Holidays API MCP Server"**, acts as a Middleware Communication Protocol (MCP) server that exposes the Canada Holidays API as a set of 6 AI agent-compatible endpoints. It enables AI agents to query Canadian public holiday information and provincial data through a structured API interface using n8n’s MCP trigger node.

**Target Use Cases:**  
- AI agents requiring Canadian holiday data for scheduling, planning, or informational purposes.  
- Developers integrating Canadian holiday data into applications without direct API coding.  
- Automations needing quick access to public holidays by province or holiday ID.

**Logical Blocks:**

- **1.1 Setup & Overview:** Workflow setup instructions and general overview sticky notes explaining usage and operations.  
- **1.2 MCP Server Trigger:** The entry point for AI agent requests, exposing a webhook URL.  
- **1.3 API Info Operations:** Nodes handling API root and schema retrieval.  
- **1.4 Holidays Operations:** Nodes that list all holidays or get details of a specific holiday by ID.  
- **1.5 Provinces Operations:** Nodes that list all provinces or get details of a specific province by ID.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Overview

- **Overview:**  
Provides setup instructions, general workflow description, and usage notes for end-users or integrators. These are all sticky notes containing textual guidance and metadata.

- **Nodes Involved:**  
  - Setup Instructions (sticky note)  
  - Workflow Overview (sticky note)

- **Node Details:**

| Node Name          | Type                   | Configuration & Role                                                                                                                          | Expressions / Variables        | Connections           | Edge Cases / Failures               | Sub-Workflow Reference |
|--------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|-----------------------|-----------------------------------|------------------------|
| Setup Instructions  | Sticky Note            | Contains detailed steps for importing, activating, using the MCP URL, and customization tips. No execution role; purely informational.        | None                          | None                  | None                              | None                   |
| Workflow Overview   | Sticky Note            | Describes the workflow purpose, API endpoints available, and how the MCP interface works.                                                    | None                          | None                  | None                              | None                   |

---

#### 1.2 MCP Server Trigger

- **Overview:**  
Acts as the webhook trigger node that listens for incoming AI agent requests. This node is the entry point of the workflow, exposing a RESTful endpoint `/canada-holidays-mcp`.

- **Nodes Involved:**  
  - Canada Holidays MCP Server (MCP Trigger node)

- **Node Details:**

| Node Name              | Type                         | Configuration & Role                                                                                     | Expressions / Variables  | Connections                             | Edge Cases / Failures                                    | Sub-Workflow Reference |
|------------------------|------------------------------|---------------------------------------------------------------------------------------------------------|-------------------------|---------------------------------------|--------------------------------------------------------|------------------------|
| Canada Holidays MCP Server | MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger) | Configured with path `canada-holidays-mcp` to expose MCP-compatible webhook. Receives all AI requests.  | None                    | Outputs connected to all HTTP request nodes | Possible webhook misconfiguration, downtime, or permission issues | None                   |

---

#### 1.3 API Info Operations

- **Overview:**  
Handles requests for API metadata: root endpoint and JSON schema, allowing AI agents to discover API structure.

- **Nodes Involved:**  
  - Sticky Note (Info)  
  - Get API Root (HTTP Request Tool)  
  - Get API Schema (HTTP Request Tool)  
  - Description - info (Sticky Note)

- **Node Details:**

| Node Name         | Type                | Configuration & Role                                                                                               | Expressions / Variables | Connections          | Edge Cases / Failures                       | Sub-Workflow Reference |
|-------------------|---------------------|-------------------------------------------------------------------------------------------------------------------|------------------------|----------------------|---------------------------------------------|------------------------|
| Sticky Note (Info) | Sticky Note         | Visual block label "Info"                                                                                          | None                   | None                 | None                                        | None                   |
| Get API Root      | HTTP Request Tool   | Performs GET request to `https://canada-holidays.ca/api/v1` to retrieve API root info. No auth required.           | None                   | Connected to MCP Trigger output | Network errors, API downtime, invalid responses | None                   |
| Get API Schema    | HTTP Request Tool   | Performs GET request to `https://canada-holidays.ca/api/v1/spec` to retrieve API schema JSON.                      | None                   | Connected to MCP Trigger output | Network errors, schema format changes             | None                   |
| Description - info | Sticky Note         | Describes the purpose of this block: to get welcome message and links.                                            | None                   | None                 | None                                        | None                   |

---

#### 1.4 Holidays Operations

- **Overview:**  
Provides endpoints to list all holidays or get detailed info for a specific holiday ID, supporting optional query parameters such as year, federal-only filter, and optional holidays flag.

- **Nodes Involved:**  
  - Sticky Note2 (Holidays)  
  - List All Holidays (HTTP Request Tool)  
  - Get Holiday by ID (HTTP Request Tool)  
  - Description - holidays (Sticky Note)

- **Node Details:**

| Node Name          | Type                | Configuration & Role                                                                                                            | Expressions / Variables                                                                                                  | Connections          | Edge Cases / Failures                                    | Sub-Workflow Reference |
|--------------------|---------------------|--------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|----------------------|----------------------------------------------------------|------------------------|
| Sticky Note2       | Sticky Note         | Visual block label "Holidays"                                                                                                  | None                                                                                                                    | None                 | None                                                     | None                   |
| List All Holidays  | HTTP Request Tool   | GET request to `https://canada-holidays.ca/api/v1/holidays` with optional query parameters: year, federal, optional. Parameters auto-populated from AI inputs using `$fromAI()`. | - `year`: number, default 2023<br>- `federal`: string boolean<br>- `optional`: string boolean, default "false"           | Connected to MCP Trigger output | API rate limits, invalid parameter types, network timeouts | None                   |
| Get Holiday by ID  | HTTP Request Tool   | GET request to `https://canada-holidays.ca/api/v1/holidays/{{ holidayId }}` with optional query parameters: year, optional. Holiday ID required, populated from AI input. | - `holidayId`: number, required<br>- `year`: number, default 2023<br>- `optional`: string boolean, default "false"        | Connected to MCP Trigger output | Missing or invalid holidayId, network errors, invalid responses | None                   |
| Description - holidays | Sticky Note         | Describes this block’s functionality: getting holidays with associated provinces.                                             | None                                                                                                                    | None                 | None                                                     | None                   |

---

#### 1.5 Provinces Operations

- **Overview:**  
Supports listing all provinces and retrieving detailed info for a specific province by its abbreviation, with optional year and optional holiday filters.

- **Nodes Involved:**  
  - Sticky Note3 (Provinces)  
  - List All Provinces (HTTP Request Tool)  
  - Get Province by ID (HTTP Request Tool)  
  - Description - provinces (Sticky Note)

- **Node Details:**

| Node Name           | Type                | Configuration & Role                                                                                                               | Expressions / Variables                                                                                           | Connections          | Edge Cases / Failures                                     | Sub-Workflow Reference |
|---------------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|----------------------|-----------------------------------------------------------|------------------------|
| Sticky Note3        | Sticky Note         | Visual block label "Provinces"                                                                                                    | None                                                                                                             | None                 | None                                                      | None                   |
| List All Provinces  | HTTP Request Tool   | GET request to `https://canada-holidays.ca/api/v1/provinces` with optional query parameters: year, optional. Parameters from AI inputs. | - `year`: number, default 2023<br>- `optional`: string boolean, default "false"                                   | Connected to MCP Trigger output | Network errors, invalid query parameters                  | None                   |
| Get Province by ID  | HTTP Request Tool   | GET request to `https://canada-holidays.ca/api/v1/provinces/{{ provinceId }}` with optional query parameters: year, optional. Province ID required, from AI input. | - `provinceId`: string, required (Canadian province abbreviation)<br>- `year`: number, default 2023<br>- `optional`: string boolean, default "false" | Connected to MCP Trigger output | Invalid provinceId, network errors                         | None                   |
| Description - provinces | Sticky Note         | Describes the block’s purpose: retrieving provinces with associated holidays.                                                      | None                                                                                                             | None                 | None                                                      | None                   |

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                               | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                                                     |
|---------------------|----------------------------------|-----------------------------------------------|----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions   | Sticky Note                      | Setup and usage instructions                   | None                       | None                         | ### ⚙️ Setup Instructions 1. Import Workflow ... [discord](https://discord.me/cfomodz), [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) |
| Workflow Overview    | Sticky Note                      | General workflow and API overview              | None                       | None                         | ## 🛠️ Canada Holidays MCP Server ... 6 endpoints                                                                             |
| Canada Holidays MCP Server | MCP Trigger                    | Webhook trigger for AI agent requests          | None                       | Connects to all HTTP Request nodes |                                                                                                                                 |
| Sticky Note (Info)   | Sticky Note                      | Label for Info operations block                 | None                       | None                         | ## Info                                                                                                                         |
| Get API Root         | HTTP Request Tool                | Get API root endpoint                           | Canada Holidays MCP Server | None                         |                                                                                                                                 |
| Get API Schema       | HTTP Request Tool                | Get API JSON schema                             | Canada Holidays MCP Server | None                         |                                                                                                                                 |
| Description - info   | Sticky Note                      | Describes Info block                            | None                       | None                         | ## 📋 Info Get welcome message and links                                                                                      |
| Sticky Note2 (Holidays) | Sticky Note                      | Label for Holidays operations block            | None                       | None                         | ## Holidays                                                                                                                     |
| List All Holidays    | HTTP Request Tool                | List all holidays with optional filters        | Canada Holidays MCP Server | None                         |                                                                                                                                 |
| Get Holiday by ID    | HTTP Request Tool                | Get holiday details by ID                        | Canada Holidays MCP Server | None                         |                                                                                                                                 |
| Description - holidays | Sticky Note                      | Describes Holidays block                        | None                       | None                         | ## 📋 Holidays Get holiday(s) with associated provinces                                                                        |
| Sticky Note3 (Provinces) | Sticky Note                      | Label for Provinces operations block            | None                       | None                         | ## Provinces                                                                                                                    |
| List All Provinces   | HTTP Request Tool                | List all provinces with optional filters        | Canada Holidays MCP Server | None                         |                                                                                                                                 |
| Get Province by ID   | HTTP Request Tool                | Get province details by abbreviation             | Canada Holidays MCP Server | None                         |                                                                                                                                 |
| Description - provinces | Sticky Note                      | Describes Provinces block                       | None                       | None                         | ## 📋 Provinces Get province(s) with associated holidays                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Setup & Overview**  
   - Add a sticky note named "Setup Instructions" with the content describing import, activation, MCP URL usage, AI parameter auto-population, customization, and support links (discord and n8n docs).  
   - Add a sticky note named "Workflow Overview" describing the workflow purpose, MCP interface, API endpoints (Info, Holidays, Provinces).

2. **Add MCP Trigger Node**  
   - Create an MCP Trigger node named "Canada Holidays MCP Server".  
   - Set the webhook path to `canada-holidays-mcp`.  
   - No authentication required.  
   - This node will serve as the entry point listening for AI agent requests.

3. **Build Info Operations Block**  
   - Add a sticky note labeled "Info" as a visual block header.  
   - Add HTTP Request Tool node named "Get API Root":  
     - Method: GET  
     - URL: `https://canada-holidays.ca/api/v1`  
     - No authentication or query parameters.  
   - Add HTTP Request Tool node named "Get API Schema":  
     - Method: GET  
     - URL: `https://canada-holidays.ca/api/v1/spec`  
     - No authentication or query parameters.  
   - Add a sticky note named "Description - info" describing this block’s purpose.  
   - Connect the output of MCP Trigger to both HTTP Request nodes.

4. **Build Holidays Operations Block**  
   - Add a sticky note labeled "Holidays".  
   - Add HTTP Request Tool node named "List All Holidays":  
     - Method: GET  
     - URL: `https://canada-holidays.ca/api/v1/holidays`  
     - Enable query parameters:  
       - `year` = `{{$fromAI('year', 'A calendar year', 'number', 2023)}}`  
       - `federal` = `{{$fromAI('federal', 'A boolean parameter. If true or 1, will return only federal holidays. If false or 0, no federal holidays.', 'string')}}`  
       - `optional` = `{{$fromAI('optional', 'A boolean parameter. If false or 0 (default), legislated holidays only. If true or 1, include optional holidays from Alberta and BC.', 'string', 'false')}}`  
   - Add HTTP Request Tool node named "Get Holiday by ID":  
     - Method: GET  
     - URL: `https://canada-holidays.ca/api/v1/holidays/{{$fromAI('holidayId', 'Primary key for a holiday', 'number')}}`  
     - Enable query parameters:  
       - `year` = `{{$fromAI('year', 'A calendar year', 'number', 2023)}}`  
       - `optional` = `{{$fromAI('optional', 'A boolean parameter. If false or 0 (default), legislated holiday provinces only. If true or 1, include optional provinces.', 'string', 'false')}}`  
   - Add a sticky note named "Description - holidays" describing this block.  
   - Connect the MCP Trigger node’s output to both HTTP Request nodes.

5. **Build Provinces Operations Block**  
   - Add a sticky note labeled "Provinces".  
   - Add HTTP Request Tool node named "List All Provinces":  
     - Method: GET  
     - URL: `https://canada-holidays.ca/api/v1/provinces`  
     - Enable query parameters:  
       - `year` = `{{$fromAI('year', 'A calendar year', 'number', 2023)}}`  
       - `optional` = `{{$fromAI('optional', 'A boolean parameter. If false or 0 (default), legislated holidays only. If true or 1, include optional holidays for AB and BC.', 'string', 'false')}}`  
   - Add HTTP Request Tool node named "Get Province by ID":  
     - Method: GET  
     - URL: `https://canada-holidays.ca/api/v1/provinces/{{$fromAI('provinceId', 'A Canadian province abbreviation', 'string')}}`  
     - Enable query parameters:  
       - `year` = `{{$fromAI('year', 'A calendar year', 'number', 2023)}}`  
       - `optional` = `{{$fromAI('optional', 'A boolean parameter (AB and BC only). If false or 0 (default), legislated holidays only. If true or 1, include optional holidays.', 'string', 'false')}}`  
   - Add a sticky note named "Description - provinces" describing this block.  
   - Connect the MCP Trigger node’s output to both HTTP Request nodes.

6. **General Settings**  
   - Set workflow timezone to `America/New_York`.  
   - No authentication credentials needed for API calls.  
   - Enable the workflow to activate MCP server.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Parameters are auto-populated by AI using `$fromAI()` expressions, allowing smooth AI agent integration without manual input.   | Usage note from Setup Instructions sticky note                                                               |
| MCP Trigger node serves as a lightweight server endpoint for AI agents to call any of the 6 API operations transparently.      | Workflow Overview sticky note                                                                                 |
| For custom needs, add data transformation, logging, or error handling nodes as per individual project requirements.             | Setup Instructions sticky note                                                                                 |
| Community support is available via Discord: https://discord.me/cfomodz and detailed n8n docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions sticky note                                                                                 |

---

**Disclaimer:** The provided documentation is based exclusively on an automated workflow built with n8n integration and automation tool. The workflow respects all content policies and manipulates only legal and public data.