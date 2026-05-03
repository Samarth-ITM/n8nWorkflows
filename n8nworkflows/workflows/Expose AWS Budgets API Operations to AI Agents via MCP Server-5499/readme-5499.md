Expose AWS Budgets API Operations to AI Agents via MCP Server

https://n8nworkflows.xyz/workflows/expose-aws-budgets-api-operations-to-ai-agents-via-mcp-server-5499


# Expose AWS Budgets API Operations to AI Agents via MCP Server

### 1. Workflow Overview

This n8n workflow, titled **"AWS Budgets MCP Server"**, exposes the Amazon Web Services (AWS) Budgets API operations to AI agents via an MCP (Modular Chatbot Protocol) server interface. It serves as a bridge between AI agents and AWS Budgets API, enabling programmatic budget management, monitoring, and alert configurations through AI interactions.

#### Target Use Cases:
- Allow AI agents to create, update, delete, and describe AWS budgets, budget actions, notifications, and subscribers.
- Provide a standardized MCP server endpoint for AI agents to invoke any of the 23 AWS Budgets API operations.
- Support AI-driven automation and monitoring of AWS cost management workflows.

#### Logical Blocks:
- **1.1 Setup and Overview**: Provides user instructions, workflow purpose, and usage notes via sticky notes.
- **1.2 MCP Server Trigger**: Single MCP trigger node that serves as the webhook endpoint receiving AI agent requests.
- **1.3 AWS Budgets API Operations**: A broad set of HTTP Request Tool nodes, each implementing a distinct AWS Budgets API operation (Create, Delete, Update, Describe, List, Execute).
- **1.4 Documentation and Grouping Notes**: Sticky notes labeling and grouping the API operation nodes by function for clarity.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Overview

**Overview:**  
This block provides setup instructions and a detailed overview of the workflow's purpose, usage, and customization options. It is purely informational, intended to guide users in configuring and utilizing the workflow.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Instructions for importing, authenticating, activating, and connecting the MCP server for AI agents.  
  - Content highlights:  
    - Setup of API Key credentials (HTTP header, key name "Authorization")  
    - Activation steps and how to get the MCP webhook URL  
    - Notes on AI usage of `$fromAI()` expressions to populate parameters  
    - Tips for customization and support links (Discord, n8n documentation)  
  - Connections: None (informational only)  
  - Edge Cases: None (static content)

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Explains the purpose of the AWS Budgets MCP Server and details the API's capabilities and how the workflow operates.  
  - Content highlights:  
    - Explanation of AWS Budgets API and budget types (Cost, Usage, RI utilization, RI coverage)  
    - Service endpoint URL: https://budgets.amazonaws.com  
    - Workflow components: MCP Trigger, HTTP Request nodes, AI expressions, native integration  
  - Connections: None (informational only)  
  - Edge Cases: None

---

#### 1.2 MCP Server Trigger

**Overview:**  
This block contains the MCP Trigger node that acts as the server endpoint receiving all AI agent requests. It listens on a designated webhook path and routes incoming requests to the appropriate API operation nodes.

**Nodes Involved:**  
- AWS Budgets MCP Server (MCP Trigger)

**Node Details:**

- **AWS Budgets MCP Server**  
  - Type: MCP Trigger (from n8n-nodes-langchain)  
  - Role: Listens for incoming AI agent requests at the webhook path `/aws-budgets-mcp`  
  - Configuration:  
    - Webhook path set to `"aws-budgets-mcp"`  
  - Connections: Outputs to all HTTP Request Tool nodes (API operation handlers) via AI tool connection  
  - Version-specific: Requires n8n version supporting MCP Trigger node (Langchain integration)  
  - Edge Cases:  
    - Webhook not activated or URL changed will cause AI agent connection failures  
    - Authentication failures if API Key is missing or invalid  
    - Request payload format errors could cause node processing failures

---

#### 1.3 AWS Budgets API Operations

**Overview:**  
This block implements all 23 AWS Budgets API operations as HTTP Request Tool nodes. Each node corresponds to one API endpoint, configured to dynamically accept parameters from AI agents via `$fromAI()` expressions, including the critical header parameter `X-Amz-Target` which specifies the AWS service operation.

**Nodes Involved:**  
- Creates a budget and, if included, notifications a  
- Creates a budget action.  
- Creates a notification. You must create the budget  
- Creates a subscriber. You must create the associat  
- Deletes a budget. You can delete your budget at an  
- Deletes a budget action.  
- Deletes a notification. Deleting a notification al  
- Deletes a subscriber. Deleting the last subscriber  
- Describes a budget. The Request Syntax section sho  
- Describes a budget action detail.  
- Describes a budget action history detail.  
- Describes all of the budget actions for an account  
- Describes all of the budget actions for a budget.  
- Lists the budget names and notifications that are   
- Describes the history for DAILY, MONTHLY, and QUAR  
- Lists the budgets that are associated with an acco  
- Lists the notifications that are associated with a  
- Lists the subscribers that are associated with a n  
- Executes a budget action.  
- Updates a budget. You can change every part of a b  
- Updates a budget action.  
- Updates a notification.  
- Updates a subscriber.

**Node Details (Typical for all HTTP Request Tool nodes):**

- Type: HTTP Request Tool  
- Role: Execute specific AWS Budgets API operations by sending POST requests to `https://budgets.amazonaws.com`, with operation specified in the `X-Amz-Target` header.  
- Configuration:  
  - Method: POST  
  - URL: `https://budgets.amazonaws.com/#X-Amz-Target=AWSBudgetServiceGateway.OperationName` dynamically constructed or fixed per node  
  - Authentication: Generic HTTP Header API Key  
  - Header Parameter:  
    - `X-Amz-Target` header populated by expression: `={{ $fromAI('X-Amz-Target', 'X Amz Target', 'string') }}` (allows AI to specify operation name)  
  - Query Parameters: Used in some nodes for pagination (`MaxResults`, `NextToken`) dynamically from AI via `$fromAI()`  
  - Description: Each node includes a tool description explaining the API endpoint purpose and important usage notes.  
- Input: MCP Trigger node (via ai_tool connection)  
- Output: Response returned to the AI agent automatically through MCP  
- Version-specific: Requires n8n version supporting HTTP Request Tool and generic HTTP Header authentication  
- Edge Cases / Potential Failures:  
  - Authentication failures (invalid/missing API key)  
  - API rate limits or throttling by AWS  
  - Malformed or incomplete parameters from AI causing API errors  
  - Network timeouts or connectivity issues  
  - Pagination token misuse causing infinite loops or missing data  
  - AWS service outages or API changes  
- Sub-workflow: None; all nodes are within this main workflow.

---

#### 1.4 Documentation and Grouping Notes

**Overview:**  
Sticky notes are used throughout the workflow to group related API operation nodes by function (e.g., Create Budget, Delete Notification), improving readability and maintainability.

**Nodes Involved:**  
- Grid Note 1 through Grid Note 23 (Sticky Notes)

**Node Details:**  
- Type: Sticky Note  
- Role: Label functional groups of API operation nodes  
- Content: Titles such as "Create Budget", "Delete Subscriber", "Describe Budget", etc.  
- Connections: None (informational only)  
- Edge Cases: None

---

### 3. Summary Table

| Node Name                                            | Node Type               | Functional Role                              | Input Node(s)           | Output Node(s) | Sticky Note                                                                                                         |
|-----------------------------------------------------|-------------------------|----------------------------------------------|-------------------------|----------------|---------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                                  | Sticky Note             | Provides setup and usage instructions        | None                    | None           | ⚙️ Setup Instructions… See [discord](https://discord.me/cfomodz) and n8n docs links for help and customization tips. |
| Workflow Overview                                   | Sticky Note             | Describes workflow purpose and AWS Budgets API | None                    | None           | Explains AWS Budgets API and workflow operation.                                                                    |
| AWS Budgets MCP Server                              | MCP Trigger             | Entry point for AI agent requests             | None                    | All HTTP Request Tool nodes |                                                                                                                     |
| Creates a budget and, if included, notifications a | HTTP Request Tool       | Creates a budget with optional notifications | AWS Budgets MCP Server   | None           | ## Create Budget                                                                                                     |
| Creates a budget action.                            | HTTP Request Tool       | Creates a budget action                       | AWS Budgets MCP Server   | None           | ## Create Budget Action                                                                                              |
| Creates a notification. You must create the budget | HTTP Request Tool       | Creates notification for a budget            | AWS Budgets MCP Server   | None           | ## Create Notification                                                                                               |
| Creates a subscriber. You must create the associat | HTTP Request Tool       | Creates subscriber for notification          | AWS Budgets MCP Server   | None           | ## Create Subscriber                                                                                                |
| Deletes a budget. You can delete your budget at an | HTTP Request Tool       | Deletes a budget and associated notifications/subscribers | AWS Budgets MCP Server   | None           | ## Delete Budget                                                                                                     |
| Deletes a budget action.                            | HTTP Request Tool       | Deletes a budget action                       | AWS Budgets MCP Server   | None           | ## Delete Budget Action                                                                                              |
| Deletes a notification. Deleting a notification al | HTTP Request Tool       | Deletes notification and its subscribers     | AWS Budgets MCP Server   | None           | ## Delete Notification                                                                                               |
| Deletes a subscriber. Deleting the last subscriber | HTTP Request Tool       | Deletes subscriber; last deletion removes notification | AWS Budgets MCP Server   | None           | ## Delete Subscriber                                                                                                |
| Describes a budget. The Request Syntax section sho | HTTP Request Tool       | Retrieves budget details                      | AWS Budgets MCP Server   | None           | ## Describe Budget                                                                                                   |
| Describes a budget action detail.                   | HTTP Request Tool       | Retrieves budget action details               | AWS Budgets MCP Server   | None           | ### Describe Budget Action                                                                                           |
| Describes a budget action history detail.           | HTTP Request Tool       | Retrieves budget action history details       | AWS Budgets MCP Server   | None           | ### Describe Budget Action Histories                                                                                 |
| Describes all of the budget actions for an account | HTTP Request Tool       | Lists all budget actions for the account     | AWS Budgets MCP Server   | None           | ### Describe Budget Actions For Account                                                                              |
| Describes all of the budget actions for a budget.  | HTTP Request Tool       | Lists all budget actions for a budget        | AWS Budgets MCP Server   | None           | ### Describe Budget Actions For Budget                                                                                |
| Lists the budget names and notifications that are  | HTTP Request Tool       | Lists budget names and notifications for account | AWS Budgets MCP Server   | None           | ### Describe Budget Notifications For Account                                                                        |
| Describes the history for DAILY, MONTHLY, and QUAR | HTTP Request Tool       | Retrieves budget performance history          | AWS Budgets MCP Server   | None           | ### Describe Budget Performance History                                                                              |
| Lists the budgets that are associated with an acco | HTTP Request Tool       | Lists budgets for an account                   | AWS Budgets MCP Server   | None           | ## Describe Budgets                                                                                                  |
| Lists the notifications that are associated with a | HTTP Request Tool       | Lists notifications for a budget               | AWS Budgets MCP Server   | None           | ### Describe Notifications For Budget                                                                                |
| Lists the subscribers that are associated with a n | HTTP Request Tool       | Lists subscribers for a notification           | AWS Budgets MCP Server   | None           | ### Describe Subscribers For Notification                                                                            |
| Executes a budget action.                           | HTTP Request Tool       | Executes a budget action                       | AWS Budgets MCP Server   | None           | ## Execute Budget Action                                                                                             |
| Updates a budget. You can change every part of a b | HTTP Request Tool       | Updates a budget                               | AWS Budgets MCP Server   | None           | ## Update Budget                                                                                                     |
| Updates a budget action.                            | HTTP Request Tool       | Updates a budget action                        | AWS Budgets MCP Server   | None           | ## Update Budget Action                                                                                              |
| Updates a notification.                             | HTTP Request Tool       | Updates a notification                         | AWS Budgets MCP Server   | None           | ## Update Notification                                                                                               |
| Updates a subscriber.                               | HTTP Request Tool       | Updates a subscriber                           | AWS Budgets MCP Server   | None           | ## Update Subscriber                                                                                                |
| Grid Note 1 - 23                                   | Sticky Note             | Grouping and labeling of API operation nodes | None                    | None           | Each sticky note groups related API operation nodes by function (Create, Delete, Describe, Update)                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Setup and Overview**  
   - Create a sticky note named **"Setup Instructions"** with the provided detailed instructions for importing, authentication, activation, and AI integration tips. Color: 4 (blueish). Height ~1060.  
   - Create a sticky note named **"Workflow Overview"** describing the AWS Budgets API purpose, endpoint, and workflow logic. Width ~320, height ~1280.

2. **Add MCP Trigger Node**  
   - Add an **MCP Trigger** node named **"AWS Budgets MCP Server"** using the `@n8n/n8n-nodes-langchain.mcpTrigger` node type.  
   - Set the webhook path to `/aws-budgets-mcp`.  
   - Leave authentication open for now; will be secured by API key headers in downstream nodes.

3. **Configure API Key Credential**  
   - Create an API Key credential in n8n settings:  
     - Authentication Type: HTTP Header  
     - Header Name: `Authorization`  
     - Key Value: (set your AWS API key or appropriate token)  

4. **Create HTTP Request Tool Nodes for each AWS Budgets API Operation**  
   For each of the 23 operations:

   - Add an **HTTP Request Tool** node.  
   - Set the HTTP method to `POST`.  
   - Set the URL to `https://budgets.amazonaws.com/#X-Amz-Target=AWSBudgetServiceGateway.OperationName`, replacing `OperationName` with the specific API operation (e.g., `CreateBudget`, `DeleteBudgetAction`).  
   - Enable sending headers.  
   - Set authentication to **Generic HTTP Header** using the API key credential created earlier.  
   - Configure the header parameter `X-Amz-Target` with the expression:  
     `={{ $fromAI('X-Amz-Target', 'X Amz Target', 'string') }}`  
     This allows dynamic operation invocation via AI parameters.  
   - Add query parameters where applicable (e.g., `MaxResults`, `NextToken`), also populated from AI expressions:  
     - `MaxResults`: `={{ $fromAI('MaxResults', 'Pagination limit', 'string') }}`  
     - `NextToken`: `={{ $fromAI('NextToken', 'Pagination token', 'string') }}`  
   - Add a descriptive tool description explaining the API operation and important notes (copy from official AWS documentation or as provided).  
   - Connect the input of each HTTP Request Tool node to the **"AWS Budgets MCP Server"** MCP Trigger node via the AI tool connection.

5. **Add Grouping Sticky Notes**  
   - Add sticky notes near related API operation nodes to label them, e.g., "Create Budget", "Delete Subscriber", "Describe Budget Action", etc., matching the original positions and color 7 (yellow/orange).  
   - Adjust sizes and positions for clarity.

6. **Activating and Testing**  
   - Activate the workflow.  
   - Copy the webhook URL from the MCP Trigger node.  
   - Configure your AI agent to send requests to this webhook URL, ensuring it sends properly structured JSON with the required parameters and the `X-Amz-Target` header value.  
   - Test calls to various AWS Budgets operations through the AI agent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automation support, contact on Discord: [discord.me/cfomodz](https://discord.me/cfomodz).                                                                                                                         | Support and community help                                                                                   |
| Refer to the n8n documentation for MCP Trigger and tool MCP usage: [n8n MCP Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)                                                                | Official n8n docs                                                                                            |
| Parameters in HTTP Request nodes are auto-populated by AI agents using the `$fromAI()` expression syntax, enabling dynamic request parameterization.                                                                                                   | Parameter handling                                                                                           |
| The workflow preserves the original AWS Budgets API response structure, facilitating direct AI consumption without transformation.                                                                                                                     | Response handling                                                                                            |
| Customize the workflow by adding data transformation, error handling, logging, or monitoring nodes as needed to fit your operational requirements.                                                                                                     | Customization tips                                                                                           |
| Use the provided grouping sticky notes to maintain clarity when modifying or extending the workflow with additional AWS Budgets API endpoints or related logic.                                                                                        | Workflow maintainability                                                                                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, a workflow automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.