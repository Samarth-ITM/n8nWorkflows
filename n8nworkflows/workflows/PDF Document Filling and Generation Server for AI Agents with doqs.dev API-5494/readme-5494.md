PDF Document Filling and Generation Server for AI Agents with doqs.dev API

https://n8nworkflows.xyz/workflows/pdf-document-filling-and-generation-server-for-ai-agents-with-doqs-dev-api-5494


# PDF Document Filling and Generation Server for AI Agents with doqs.dev API

### 1. Workflow Overview

This workflow implements a **PDF Document Filling and Generation Server** using the doqs.dev API, exposing a comprehensive set of 14 PDF-related operations as an MCP (Model-Callable-Plugin) interface for AI agents. It acts as a server endpoint that listens for AI agent requests, forwards those requests to the doqs.dev API via HTTP requests, and returns the responses directly to the calling AI agents.

The workflow is logically segmented into:

- **1.1 MCP Trigger Endpoint:** The entry point serving as the webhook endpoint for AI agent requests.
- **1.2 Designer Operations Block:** Handles 7 doqs.dev API operations related to PDF template design and management.
- **1.3 Templates Operations Block:** Handles 7 doqs.dev API operations related to PDF template usage and document generation/filling.

Each block contains HTTP Request nodes configured to automatically receive parameters from AI agents via `$fromAI()` expressions, enabling dynamic request handling with minimal manual intervention.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Endpoint

- **Overview:**  
  Serves as the main webhook entry point for AI agents to invoke the PDF filling and generation operations. It listens on a specific path and routes the requests to the appropriate HTTP Request nodes based on the operation requested.

- **Nodes Involved:**  
  - `doqs.dev | PDF filling MCP Server`

- **Node Details:**  
  - **Type & Role:** MCP Trigger node; serves as a server endpoint compatible with AI agents using MCP protocol.  
  - **Configuration:**  
    - Webhook path: `doqs.dev-|-pdf-filling-mcp`  
    - Exposes 14 API operations as callable tools.  
  - **Expressions:** Uses AI context variables implicitly via MCP mechanism.  
  - **Connections:** Outputs to all HTTP Request nodes in Designer and Templates blocks via `ai_tool` connections.  
  - **Edge Cases:**  
    - Webhook request failures (e.g., invalid payload or unauthorized requests).  
    - Missing or incorrect API key credentials can cause authentication failures downstream.  
  - **Version Requirements:** Compatible with n8n MCP integrations (requires n8n versions supporting MCP triggers and httpRequestTool nodes).

---

#### 2.2 Designer Operations Block

- **Overview:**  
  Manages 7 API endpoints under the "Designer" domain of doqs.dev, which include listing, creating, updating, previewing, deleting, and generating PDFs from templates. This block enables full lifecycle management of PDF templates in the design phase.

- **Nodes Involved:**  
  - Sticky Note (label: "Designer")  
  - List Templates  
  - Create Template  
  - Preview  
  - Delete  
  - List Templates 1 (Get Template by ID)  
  - Update Template  
  - Generate Pdf

- **Node Details:**

  1. **List Templates**  
     - Type: HTTP Request Tool  
     - Role: Retrieves paginated list of designer templates.  
     - Config: GET `https://api.doqs.dev/v1/designer/templates/` with optional query params `limit` and `offset` dynamically populated using `$fromAI('limit', 'Limit', 'number', 100)` and `$fromAI('offset', 'Offset', 'number', 0)`.  
     - Input: MCP Trigger  
     - Output: Returns JSON list of templates.  
     - Edge Cases: Invalid query params, network issues.

  2. **Create Template**  
     - Type: HTTP Request Tool  
     - Role: Creates a new designer template.  
     - Config: POST `https://api.doqs.dev/v1/designer/templates/` with request body populated from AI context.  
     - Edge Cases: Missing required body parameters, validation errors.

  3. **Preview**  
     - Type: HTTP Request Tool  
     - Role: Generates a preview of a designer template.  
     - Config: POST `https://api.doqs.dev/v1/designer/templates/preview` with template data from AI.  
     - Edge Cases: Invalid template data, API errors.

  4. **Delete**  
     - Type: HTTP Request Tool  
     - Role: Deletes a designer template by ID.  
     - Config: DELETE `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}`  
     - Edge Cases: Non-existent ID, permission errors.

  5. **List Templates 1**  
     - Type: HTTP Request Tool  
     - Role: Retrieves a specific designer template by ID.  
     - Config: GET `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}`  
     - Edge Cases: Invalid or missing ID.

  6. **Update Template**  
     - Type: HTTP Request Tool  
     - Role: Updates a designer template by ID.  
     - Config: PUT `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}` with update data.  
     - Edge Cases: Validation errors, missing ID.

  7. **Generate Pdf**  
     - Type: HTTP Request Tool  
     - Role: Generates a PDF from a designer template ID.  
     - Config: POST `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}/generate`  
     - Edge Cases: Invalid ID, generation failures.

---

#### 2.3 Templates Operations Block

- **Overview:**  
  Manages 7 API endpoints related to template instances for filling, retrieving files, and CRUD operations on templates in active use. This block allows AI agents to interact with templates for document generation and filling.

- **Nodes Involved:**  
  - Sticky Note (label: "Templates")  
  - List  
  - Create  
  - Delete   
  - Get Template  
  - Update  
  - Get File  
  - Fill

- **Node Details:**

  1. **List**  
     - Type: HTTP Request Tool  
     - Role: Lists templates with optional pagination.  
     - Config: GET `https://api.doqs.dev/v1/templates` with query params `limit` and `offset` populated from AI context.  
     - Edge Cases: Pagination errors.

  2. **Create**  
     - Type: HTTP Request Tool  
     - Role: Creates a new template instance.  
     - Config: POST `https://api.doqs.dev/v1/templates` with request body from AI.  
     - Edge Cases: Missing required fields.

  3. **Delete**  
     - Type: HTTP Request Tool  
     - Role: Deletes a template instance by ID.  
     - Config: DELETE `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}`  
     - Edge Cases: Invalid ID.

  4. **Get Template**  
     - Type: HTTP Request Tool  
     - Role: Retrieves a template instance by ID.  
     - Config: GET `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}`  
     - Edge Cases: Missing or invalid ID.

  5. **Update**  
     - Type: HTTP Request Tool  
     - Role: Updates a template instance by ID.  
     - Config: PUT `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}` with update body.  
     - Edge Cases: Validation errors.

  6. **Get File**  
     - Type: HTTP Request Tool  
     - Role: Retrieves the file associated with a template instance by ID.  
     - Config: GET `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}/file`  
     - Edge Cases: File not found.

  7. **Fill**  
     - Type: HTTP Request Tool  
     - Role: Fills a template with provided data by ID.  
     - Config: POST `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}/fill` with fill data.  
     - Edge Cases: Invalid data, fill failures.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                                 | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                                          |
|-------------------------|---------------------------|------------------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview       | Sticky Note               | Describes workflow purpose and instructions    |                              |                               | Contains overview, setup instructions, usage notes, and customization tips for the entire workflow                                   |
| doqs.dev | PDF filling MCP Server | MCP Trigger               | Entry point for AI agent requests               | All HTTP Request nodes        |                               | Serves as MCP server endpoint; exposes 14 doqs.dev API operations                                                                  |
| Sticky Note             | Sticky Note               | Labels Designer operations block                |                              |                               | Label: "Designer"                                                                                                                     |
| List Templates          | HTTP Request Tool         | List designer templates                          | MCP Trigger                  |                               |                                                                                                                                    |
| Create Template         | HTTP Request Tool         | Create a new designer template                   | MCP Trigger                  |                               |                                                                                                                                    |
| Preview                 | HTTP Request Tool         | Preview a designer template                      | MCP Trigger                  |                               |                                                                                                                                    |
| Delete                  | HTTP Request Tool         | Delete a designer template by ID                 | MCP Trigger                  |                               |                                                                                                                                    |
| List Templates 1        | HTTP Request Tool         | Get a designer template by ID                    | MCP Trigger                  |                               |                                                                                                                                    |
| Update Template         | HTTP Request Tool         | Update a designer template by ID                 | MCP Trigger                  |                               |                                                                                                                                    |
| Generate Pdf            | HTTP Request Tool         | Generate PDF from designer template ID           | MCP Trigger                  |                               |                                                                                                                                    |
| Sticky Note2            | Sticky Note               | Labels Templates operations block                |                              |                               | Label: "Templates"                                                                                                                   |
| List                    | HTTP Request Tool         | List template instances                           | MCP Trigger                  |                               |                                                                                                                                    |
| Create                  | HTTP Request Tool         | Create a new template instance                    | MCP Trigger                  |                               |                                                                                                                                    |
| Delete                  | HTTP Request Tool         | Delete template instance by ID                    | MCP Trigger                  |                               |                                                                                                                                    |
| Get Template            | HTTP Request Tool         | Get template instance by ID                       | MCP Trigger                  |                               |                                                                                                                                    |
| Update                  | HTTP Request Tool         | Update template instance by ID                    | MCP Trigger                  |                               |                                                                                                                                    |
| Get File                | HTTP Request Tool         | Retrieve file of a template instance by ID       | MCP Trigger                  |                               |                                                                                                                                    |
| Fill                    | HTTP Request Tool         | Fill a template instance with data by ID         | MCP Trigger                  |                               |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `doqs.dev | PDF filling MCP Server`  
   - Parameters:  
     - Path: `doqs.dev-|-pdf-filling-mcp`  
   - This node will serve as the webhook entry point.

2. **Set up Credentials**  
   - Configure an API Key credential with:  
     - Authentication type: API Key in HTTP Header  
     - Header Key Name: `x-api-key`  
     - Value: your doqs.dev API key

3. **Create Designer Block HTTP Request Nodes**  
   For each node below, set Authentication to the credential configured above and use `HTTP Request Tool` node type with the configured URL, method, and parameters:

   3.1 **List Templates**  
      - GET `https://api.doqs.dev/v1/designer/templates/`  
      - Query params:  
        - `limit` from AI: `$fromAI('limit', 'Limit', 'number', 100)`  
        - `offset` from AI: `$fromAI('offset', 'Offset', 'number', 0)`

   3.2 **Create Template**  
      - POST `https://api.doqs.dev/v1/designer/templates/`  
      - Body: populated dynamically from AI input

   3.3 **Preview**  
      - POST `https://api.doqs.dev/v1/designer/templates/preview`  
      - Body: populated from AI

   3.4 **Delete**  
      - DELETE `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}`

   3.5 **List Templates 1** (Get Template by ID)  
      - GET `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}`

   3.6 **Update Template**  
      - PUT `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}`  
      - Body: provided by AI

   3.7 **Generate Pdf**  
      - POST `https://api.doqs.dev/v1/designer/templates/{{ $fromAI('id', 'Id', 'string') }}/generate`

4. **Create Templates Block HTTP Request Nodes**

   4.1 **List**  
      - GET `https://api.doqs.dev/v1/templates`  
      - Query params:  
        - `limit` from AI: `$fromAI('limit', 'Limit', 'number', 100)`  
        - `offset` from AI: `$fromAI('offset', 'Offset', 'number', 0)`

   4.2 **Create**  
      - POST `https://api.doqs.dev/v1/templates`  
      - Body from AI

   4.3 **Delete**  
      - DELETE `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}`

   4.4 **Get Template**  
      - GET `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}`

   4.5 **Update**  
      - PUT `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}`  
      - Body from AI

   4.6 **Get File**  
      - GET `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}/file`

   4.7 **Fill**  
      - POST `https://api.doqs.dev/v1/templates/{{ $fromAI('id', 'Id', 'string') }}/fill`  
      - Body from AI

5. **Connect all HTTP Request nodes to the MCP Trigger node**  
   - Use `ai_tool` type connections from MCP Trigger to each HTTP Request node. This enables AI agents to call any of these operations through the single MCP endpoint.

6. **Add Sticky Notes for clarity**  
   - Place one sticky note labeled "Designer" near the Designer block nodes.  
   - Place another sticky note labeled "Templates" near the Templates block nodes.  
   - Add a large sticky note for overall workflow overview and instructions.

7. **Activate Workflow**  
   - After configuring all nodes and credentials, activate the workflow.  
   - Copy the webhook URL from the MCP Trigger node for AI agent integration.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                                             |
|------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow exposes the doqs.dev PDF filling API as an MCP server with 14 operations to AI agents.             | Workflow overview sticky note in the workflow                                                                                |
| Setup requires an API key credential with header key `x-api-key`.                                               | See "Setup Instructions" in the workflow overview sticky note                                                                |
| Parameters are dynamically populated using `$fromAI()` expressions for seamless AI integration.                 | Workflow overview and HTTP Request Tool node configurations                                                                  |
| For MCP integration help or customization, refer to n8n documentation or contact via Discord: https://discord.me/cfomodz | Workflow overview sticky note                                                                                                |
| Documentation for n8n MCP triggers: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Helpful resource linked in sticky notes                                                                                      |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.