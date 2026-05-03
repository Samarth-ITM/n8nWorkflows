Complete eBay Feed API Integration for AI Agents with MCP Server

https://n8nworkflows.xyz/workflows/complete-ebay-feed-api-integration-for-ai-agents-with-mcp-server-5578


# Complete eBay Feed API Integration for AI Agents with MCP Server

### 1. Workflow Overview

This workflow, titled **"Complete eBay Feed API Integration for AI Agents with MCP Server"**, is designed to fully integrate the eBay Feed API capabilities within an n8n environment, orchestrated through an MCP (Managed Control Plane) Server node tailored for AI agents. Its core purpose is to enable AI-driven automation and management of eBay feed-related tasks by interfacing with various eBay Feed API endpoints, allowing for creation, retrieval, updating, deletion, and downloading of feed-related data such as customer service metrics, inventory, orders, schedules, and tasks.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and MCP Server Trigger:** Receives incoming requests for feed operations via the MCP Server node designed for AI agents.
- **1.2 Customer Service Metric Tasks Management:** Handles listing, creation, and retrieval of customer service metric tasks.
- **1.3 Inventory Tasks Management:** Manages listing, creation, and retrieval of inventory-related tasks.
- **1.4 Order Tasks Management:** Facilitates listing, creation, and retrieval of order-related tasks.
- **1.5 Schedule Management:** Covers listing, creation, deletion, updating, retrieving schedules, downloading schedule result files, and managing schedule templates.
- **1.6 General Task Management:** Provides functionality for listing tasks, creating tasks, retrieving tasks, downloading input/result files, and uploading task files.

Each block consists predominantly of HTTP Request Tool nodes configured to interact with the corresponding eBay API endpoints, all invoked via the MCP Server trigger node that acts as an AI tool webhook endpoint.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and MCP Server Trigger

**Overview:**  
This block initializes the workflow by receiving API calls or commands from AI agents through the MCP Server node, acting as the central entry point for all feed-related operations.

**Nodes Involved:**  
- Feed MCP Server  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Sticky Note (positioned near MCP Server)

**Node Details:**  

- **Feed MCP Server**  
  - Type: MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
  - Role: Acts as webhook trigger to receive requests from AI agents for feed API interactions  
  - Configuration: No additional parameters; uses a webhook ID for external invocation  
  - Inputs: External webhook calls  
  - Outputs: Passes data and commands to subsequent HTTP Request nodes  
  - Failure Modes: Webhook timeouts, malformed requests, connectivity issues  
  - Version: Requires n8n version supporting MCP Trigger node  
  - Notes: Central node for AI agent integration; all HTTP Request nodes depend on it for invocation

- **Setup Instructions, Workflow Overview, Sticky Note (all Sticky Note nodes)**  
  - Type: Sticky Note  
  - Role: Documentation placeholders within the workflow for setup and overview guidance  
  - Configuration: Blank content; serve as in-editor comments  
  - No inputs or outputs; purely informational  

---

#### 1.2 Customer Service Metric Tasks Management

**Overview:**  
Handles operations related to eBay customer service metric tasks, including listing existing tasks, creating new tasks, and retrieving specific task details.

**Nodes Involved:**  
- List Customer Service Metric Tasks  
- Create Customer Service Metric Task  
- Get Customer Service Metric Task  
- Sticky Note2

**Node Details:**  

- **List Customer Service Metric Tasks**  
  - Type: HTTP Request Tool  
  - Role: Sends GET requests to retrieve a list of customer service metric tasks from eBay Feed API  
  - Configuration: Configured with endpoint URL, authentication credentials (likely OAuth2 or API key for eBay), and HTTP method GET  
  - Inputs: Triggered by Feed MCP Server node  
  - Outputs: Returns JSON list of tasks  
  - Failure Modes: Authentication errors, API rate limits, network timeouts

- **Create Customer Service Metric Task**  
  - Type: HTTP Request Tool  
  - Role: Sends POST requests to create a new customer service metric task  
  - Configuration: Includes payload body with task parameters, authentication, and endpoint URL  
  - Inputs: Triggered by MCP Server  
  - Outputs: Response includes task creation confirmation and task ID  
  - Failure Modes: Invalid payload, authentication failure, API validation errors

- **Get Customer Service Metric Task**  
  - Type: HTTP Request Tool  
  - Role: Retrieves details of a specific customer service metric task by task ID  
  - Configuration: Uses dynamic parameters for task ID, GET method, authentication  
  - Inputs: Triggered by MCP Server, expects task ID parameter  
  - Outputs: JSON object with task details  
  - Failure Modes: Missing or incorrect task ID, authorization errors

- **Sticky Note2**  
  - Role: Contextual note for this block, content empty but positioned to document Customer Service Metric Task handling

---

#### 1.3 Inventory Tasks Management

**Overview:**  
Manages eBay inventory-related feed tasks, allowing listing, creation, and retrieval of inventory tasks.

**Nodes Involved:**  
- List Inventory Tasks  
- Create Inventory Task  
- Get Inventory Task  
- Sticky Note3

**Node Details:**  

- **List Inventory Tasks**  
  - Type: HTTP Request Tool  
  - Role: Retrieves a list of inventory tasks via GET request  
  - Configuration: Endpoint URL, authentication, GET method  
  - Inputs: MCP Server node  
  - Outputs: JSON list of inventory tasks  
  - Failure Modes: Network issues, invalid credentials

- **Create Inventory Task**  
  - Type: HTTP Request Tool  
  - Role: Creates a new inventory task with POST request and payload specifying inventory details  
  - Inputs: MCP Server  
  - Outputs: Confirmation and task ID  
  - Failure Modes: Payload errors, auth failures

- **Get Inventory Task**  
  - Type: HTTP Request Tool  
  - Role: Retrieves details for a specific inventory task by ID  
  - Inputs: MCP Server with task ID parameter  
  - Outputs: Task details JSON  
  - Failure Modes: Invalid ID, permission issues

- **Sticky Note3**  
  - Role: Placeholder note for Inventory Task section

---

#### 1.4 Order Tasks Management

**Overview:**  
Facilitates operations on order-related feed tasks including listing, creation, and retrieval.

**Nodes Involved:**  
- List Order Tasks  
- Create Order Task  
- Get Order Task  
- Sticky Note4

**Node Details:**  

- **List Order Tasks**  
  - Type: HTTP Request Tool  
  - Role: Lists order tasks via GET request  
  - Inputs: Triggered by MCP Server  
  - Outputs: JSON array of order tasks  
  - Failure Modes: API access errors

- **Create Order Task**  
  - Type: HTTP Request Tool  
  - Role: Posts a new order task creation request  
  - Inputs: MCP Server  
  - Outputs: Task creation confirmation  
  - Failure Modes: Validation errors

- **Get Order Task**  
  - Type: HTTP Request Tool  
  - Role: Retrieves specific order task details  
  - Inputs: MCP Server with task ID  
  - Outputs: Task JSON data  
  - Failure Modes: Authorization, not found

- **Sticky Note4**  
  - Role: Section note for Order Tasks

---

#### 1.5 Schedule Management

**Overview:**  
Manages feed schedules including listing, creating, deleting, updating, retrieving schedules and templates, and downloading schedule result files.

**Nodes Involved:**  
- List Schedules  
- Create Schedule 1  
- Delete Schedule 1  
- Get Schedule 3  
- Update Schedule 1  
- Download Schedule Result File  
- List Schedule Templates  
- Get Schedule Template  
- Sticky Note5

**Node Details:**  

- **List Schedules**  
  - HTTP GET to retrieve schedules  
- **Create Schedule 1**  
  - HTTP POST to create new schedule  
- **Delete Schedule 1**  
  - HTTP DELETE to remove schedule  
- **Get Schedule 3**  
  - HTTP GET for schedule details  
- **Update Schedule 1**  
  - HTTP PUT/PATCH to update schedule  
- **Download Schedule Result File**  
  - HTTP GET to download results file  
- **List Schedule Templates**  
  - HTTP GET to list available schedule templates  
- **Get Schedule Template**  
  - HTTP GET to retrieve specific template info  

All nodes are HTTP Request Tool nodes configured with appropriate endpoints, authentication, and parameters, triggered by the MCP Server node.

Failure modes include authentication failures, invalid IDs, network issues, and API response errors.

- **Sticky Note5**  
  - Contextual note for Schedule management

---

#### 1.6 General Task Management

**Overview:**  
Supports general task operations such as listing all tasks, creating tasks, retrieving task details, downloading input/result files, and uploading task files.

**Nodes Involved:**  
- List Tasks  
- Create Task 2  
- Get Task 4  
- Download Task Input File  
- Download Task Result File  
- Upload Task File

**Node Details:**  

- **List Tasks**  
  - GET request to list all tasks  
- **Create Task 2**  
  - POST request to create a new task  
- **Get Task 4**  
  - GET request for specific task details  
- **Download Task Input File**  
  - GET request to download input file associated with a task  
- **Download Task Result File**  
  - GET request to download result file from a task  
- **Upload Task File**  
  - POST request to upload a file for a task  

All configured with authentication and appropriate parameters, triggered by the MCP Server node.

Failure modes: authentication, file handling errors, invalid task IDs.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                           | Input Node(s)      | Output Node(s)     | Sticky Note                              |
|-------------------------------|---------------------------|------------------------------------------|--------------------|--------------------|-----------------------------------------|
| Setup Instructions            | Sticky Note               | Setup guidance (empty content)           | None               | None               |                                         |
| Workflow Overview            | Sticky Note               | Overview guidance (empty content)        | None               | None               |                                         |
| Feed MCP Server              | MCP Trigger               | Central trigger for AI agent requests    | External webhook   | All HTTP Request nodes |                                         |
| Sticky Note                  | Sticky Note               | Context note near MCP Server              | None               | None               |                                         |
| List Customer Service Metric Tasks | HTTP Request Tool        | List customer service metric tasks       | Feed MCP Server    | -                  |                                         |
| Create Customer Service Metric Task | HTTP Request Tool        | Create customer service metric task      | Feed MCP Server    | -                  |                                         |
| Get Customer Service Metric Task | HTTP Request Tool        | Retrieve customer service metric task    | Feed MCP Server    | -                  |                                         |
| Sticky Note2                 | Sticky Note               | Customer Service Metric Tasks section note | None               | None               |                                         |
| List Inventory Tasks         | HTTP Request Tool         | List inventory tasks                      | Feed MCP Server    | -                  |                                         |
| Create Inventory Task        | HTTP Request Tool         | Create inventory task                     | Feed MCP Server    | -                  |                                         |
| Get Inventory Task           | HTTP Request Tool         | Get inventory task details                | Feed MCP Server    | -                  |                                         |
| Sticky Note3                 | Sticky Note               | Inventory Tasks section note              | None               | None               |                                         |
| List Order Tasks             | HTTP Request Tool         | List order tasks                          | Feed MCP Server    | -                  |                                         |
| Create Order Task            | HTTP Request Tool         | Create order task                         | Feed MCP Server    | -                  |                                         |
| Get Order Task               | HTTP Request Tool         | Get order task details                    | Feed MCP Server    | -                  |                                         |
| Sticky Note4                 | Sticky Note               | Order Tasks section note                  | None               | None               |                                         |
| List Schedules               | HTTP Request Tool         | List schedules                           | Feed MCP Server    | -                  |                                         |
| Create Schedule 1            | HTTP Request Tool         | Create schedule                          | Feed MCP Server    | -                  |                                         |
| Delete Schedule 1            | HTTP Request Tool         | Delete schedule                          | Feed MCP Server    | -                  |                                         |
| Get Schedule 3               | HTTP Request Tool         | Get schedule details                     | Feed MCP Server    | -                  |                                         |
| Update Schedule 1            | HTTP Request Tool         | Update schedule                         | Feed MCP Server    | -                  |                                         |
| Download Schedule Result File | HTTP Request Tool        | Download schedule result file            | Feed MCP Server    | -                  |                                         |
| List Schedule Templates      | HTTP Request Tool         | List schedule templates                   | Feed MCP Server    | -                  |                                         |
| Get Schedule Template        | HTTP Request Tool         | Get schedule template details             | Feed MCP Server    | -                  |                                         |
| Sticky Note5                 | Sticky Note               | Schedule Management section note          | None               | None               |                                         |
| List Tasks                  | HTTP Request Tool         | List all tasks                           | Feed MCP Server    | -                  |                                         |
| Create Task 2               | HTTP Request Tool         | Create new task                          | Feed MCP Server    | -                  |                                         |
| Get Task 4                  | HTTP Request Tool         | Get task details                        | Feed MCP Server    | -                  |                                         |
| Download Task Input File     | HTTP Request Tool         | Download input file for task             | Feed MCP Server    | -                  |                                         |
| Download Task Result File    | HTTP Request Tool         | Download result file for task            | Feed MCP Server    | -                  |                                         |
| Upload Task File            | HTTP Request Tool         | Upload file to task                      | Feed MCP Server    | -                  |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**  
   - Add node: **Feed MCP Server**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure webhook ID or use default webhook setup  
   - No parameters needed  
   - This node will serve as the trigger for all subsequent operations

2. **Add Sticky Notes for Documentation:**  
   - Add **Setup Instructions** and **Workflow Overview** sticky notes with your own descriptive content  
   - Place near the MCP Server node for clarity

3. **Create Customer Service Metric Task Nodes:**  
   - Add three HTTP Request nodes named:  
     - List Customer Service Metric Tasks  
     - Create Customer Service Metric Task  
     - Get Customer Service Metric Task  
   - Configure each with appropriate HTTP method (GET for list and get, POST for create)  
   - Set URL endpoints according to eBay Feed API documentation for customer service metric tasks  
   - Set authentication credentials (OAuth2 or API key for eBay)  
   - Use expressions to dynamically pass task IDs where needed (e.g., in Get node)  
   - Connect each node’s input from **Feed MCP Server**

4. **Add Inventory Task Nodes:**  
   - Similarly add List, Create, and Get Inventory Task HTTP Request nodes  
   - Configure endpoints, methods, authentication  
   - Connect inputs from MCP Server

5. **Add Order Task Nodes:**  
   - Add List, Create, and Get Order Task HTTP Request nodes  
   - Configure as above and connect inputs from MCP Server

6. **Add Schedule Management Nodes:**  
   - Add List Schedules, Create Schedule 1, Delete Schedule 1, Get Schedule 3, Update Schedule 1, Download Schedule Result File, List Schedule Templates, Get Schedule Template nodes  
   - Configure with respective HTTP methods (GET, POST, DELETE, PUT/PATCH) and endpoint URLs  
   - Set authentication  
   - Connect inputs from MCP Server

7. **Add General Task Management Nodes:**  
   - Add List Tasks, Create Task 2, Get Task 4, Download Task Input File, Download Task Result File, Upload Task File nodes  
   - Configure each with appropriate HTTP methods and endpoints  
   - Setup authentication  
   - Connect inputs from MCP Server

8. **Setup Credentials:**  
   - Configure eBay API credentials using OAuth2 or API key credentials in n8n’s credentials manager  
   - Assign these credentials to each HTTP Request node accordingly

9. **Expressions and Parameters:**  
   - Use n8n expressions to dynamically extract parameters from the incoming MCP Server webhook payload, e.g., task IDs, schedule IDs, file IDs  
   - Map these expressions into HTTP Request URLs and request bodies as required

10. **Testing:**  
    - Test each API operation independently by triggering the MCP Server webhook and passing appropriate parameters  
    - Handle error responses and timeouts by enabling retry or error workflow branches as needed

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is designed for AI agent integration via the MCP Server node to manage eBay feeds.| MCP (Managed Control Plane) Server integration is key to AI-driven workflow triggers.             |
| Use official eBay Feed API documentation to configure HTTP Request nodes precisely.             | https://developer.ebay.com/api-docs/commerce/feed/overview.html                                   |
| Ensure OAuth2 credentials are correctly set up and periodically refreshed to avoid auth failures.| eBay API requires OAuth2 tokens for most operations.                                              |
| Sticky notes within the workflow serve as placeholders for in-editor documentation and references.| They do not affect execution but are useful for maintenance and knowledge transfer.               |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a powerful integration and automation tool. This content strictly adheres to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.