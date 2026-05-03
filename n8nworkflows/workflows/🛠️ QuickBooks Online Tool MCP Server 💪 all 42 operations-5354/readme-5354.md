🛠️ QuickBooks Online Tool MCP Server 💪 all 42 operations

https://n8nworkflows.xyz/workflows/----quickbooks-online-tool-mcp-server----all-42-operations-5354


# 🛠️ QuickBooks Online Tool MCP Server 💪 all 42 operations

### 1. Workflow Overview

This workflow, titled **"QuickBooks Online Tool MCP Server"**, is designed to serve as a comprehensive multi-command processing (MCP) server for QuickBooks Online operations within n8n. It exposes a single MCP trigger node that can handle 42 distinct QuickBooks Online operations, each represented by an individual QuickBooks Tool node. The workflow is structured to receive commands and parameters from external sources via the MCP trigger, execute the corresponding QuickBooks Online API operation, and return results.

**Target Use Cases:**  
- Centralized QuickBooks Online API interaction hub supporting all common entity CRUD (Create, Read, Update, Delete) and action operations.  
- Enables integration platforms or AI agents to call specific QuickBooks operations dynamically via a single webhook endpoint.  
- Useful for automating accounting workflows, syncing data, or building custom QuickBooks applications with n8n as backend.

**Logical Blocks:**  
The workflow is logically divided into the following blocks based on QuickBooks entities and their operations:  

- **1.1 MCP Input Reception:** Single entry point for MCP commands via LangChain MCP Trigger node.  
- **1.2 Bills Operations:** Create, Get, Get many, Update, Delete bills.  
- **1.3 Customers Operations:** Create, Get, Get many, Update customers.  
- **1.4 Employees Operations:** Create, Get, Get many, Update employees.  
- **1.5 Estimates Operations:** Create, Get, Get many, Update, Send, Delete estimates.  
- **1.6 Invoices Operations:** Create, Get, Get many, Update, Send, Delete, Void invoices.  
- **1.7 Items Operations:** Get, Get many items.  
- **1.8 Payments Operations:** Create, Get, Get many, Update, Send, Delete, Void payments.  
- **1.9 Purchases Operations:** Get, Get many purchases.  
- **1.10 Reports Operations:** Get reports.  
- **1.11 Vendors Operations:** Create, Get, Get many, Update vendors.  

Each block contains QuickBooks Tool nodes that perform the respective API calls, all linked back to the central MCP trigger node as the input source.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Input Reception

**Overview:**  
This block acts as the single entry point to the workflow. It receives incoming MCP commands (via LangChain MCP Trigger) that specify which QuickBooks operation to perform along with any required parameters.

**Nodes Involved:**  
- QuickBooks Online Tool MCP Server (MCP Trigger)

**Node Details:**  
- **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
- **Role:** Entry webhook for multi-command processing requests.  
- **Configuration:** Uses a fixed webhook ID to listen; no parameters configured internally.  
- **Inputs:** External webhook requests.  
- **Outputs:** Routes data to all QuickBooks Tool nodes via the `ai_tool` output connection.  
- **Version Requirements:** Requires n8n with LangChain MCP Trigger node support.  
- **Potential Failures:** Webhook connectivity issues, payload validation errors, missing or malformed commands.  
- **Sub-workflow:** None.

---

#### 1.2 Bills Operations

**Overview:**  
This block handles all bill-related operations, including creating, retrieving (single or multiple), updating, and deleting bills in QuickBooks Online.

**Nodes Involved:**  
- Create a bill  
- Delete a bill  
- Get a bill  
- Get many bills  
- Update a bill

**Node Details:**  
Each node is of type `quickbooksTool` configured for the respective bill operation. They receive input from the MCP Trigger node and output results back to it.

- **Type:** `n8n-nodes-base.quickbooksTool`  
- **Role:** Perform bill operations via QuickBooks Online API.  
- **Configuration:** No explicit parameters in workflow JSON; dynamically receives operation and parameters from MCP Trigger.  
- **Inputs:** MCP Trigger node output.  
- **Outputs:** MCP Trigger node input (as response).  
- **Version Requirements:** Requires valid QuickBooks OAuth2 credentials configured in n8n.  
- **Potential Failures:** Authentication errors, invalid bill IDs, API rate limiting, network timeouts.  
- **Sticky Notes:** None with content.  
- **Sub-workflow:** None.

---

#### 1.3 Customers Operations

**Overview:**  
Handles creating, retrieving (single and multiple), and updating customer records in QuickBooks Online.

**Nodes Involved:**  
- Create a customer  
- Get a customer  
- Get many customers  
- Update a customer

**Node Details:**  
All nodes are QuickBooks Tool nodes configured for customer entity operations, connected similarly to the MCP Trigger.

- **Type:** `quickbooksTool`  
- **Role:** Manage customer data in QuickBooks.  
- **Configuration:** Dynamic via MCP input.  
- **Inputs/Outputs:** Connected to MCP trigger.  
- **Potential Failures:** Auth issues, invalid customer IDs, data validation errors.  
- **Sticky Notes:** Present but empty content.

---

#### 1.4 Employees Operations

**Overview:**  
Manages employee data CRUD operations.

**Nodes Involved:**  
- Create an employee  
- Get an employee  
- Get many employees  
- Update an employee

**Node Details:**  
Similar configuration and role as previous blocks, dedicated to employee entity.

- **Potential Failures:** Same as prior blocks, plus possible data schema mismatches.  
- **Sticky Notes:** Present but with no content.

---

#### 1.5 Estimates Operations

**Overview:**  
Handles estimate creation, retrieval, updating, sending, and deletion.

**Nodes Involved:**  
- Create an estimate  
- Delete an estimate  
- Get an estimate  
- Get many estimates  
- Send an estimate  
- Update an estimate

**Node Details:**  
QuickBooks Tool nodes configured for estimate operations.

- **Potential Failures:** Invalid estimate IDs, sending failures due to email issues.  
- **Sticky Notes:** Present but empty.

---

#### 1.6 Invoices Operations

**Overview:**  
Supports full invoice lifecycle: create, get, get many, update, send, delete, and void.

**Nodes Involved:**  
- Create an invoice  
- Delete an invoice  
- Get an invoice  
- Get many invoices  
- Send an invoice  
- Update an invoice  
- Void an invoice

**Node Details:**  
Configured QuickBooks Tool nodes handle invoice API calls.

- **Potential Failures:** Voiding restrictions, send email errors, invalid IDs.  
- **Sticky Notes:** Present but no content.

---

#### 1.7 Items Operations

**Overview:**  
Provides retrieval of single or multiple inventory or service items.

**Nodes Involved:**  
- Get an item  
- Get many items

**Node Details:**  
QuickBooks Tool nodes for item lookup.

- **Potential Failures:** Non-existent item IDs, API limits.  
- **Sticky Notes:** Present but empty.

---

#### 1.8 Payments Operations

**Overview:**  
Manages payments including create, get, get many, update, send, delete, and void.

**Nodes Involved:**  
- Create a payment  
- Delete a payment  
- Get a payment  
- Get many payments  
- Send a payment  
- Update a payment  
- Void a payment

**Node Details:**  
QuickBooks Tool nodes for payment entity.

- **Potential Failures:** Payment void restrictions, invalid IDs, sending failures.  
- **Sticky Notes:** Present but empty.

---

#### 1.9 Purchases Operations

**Overview:**  
Supports retrieval of purchase records.

**Nodes Involved:**  
- Get a purchase  
- Get many purchases

**Node Details:**  
QuickBooks Tool nodes to fetch purchases.

- **Potential Failures:** Invalid purchase IDs, API errors.  
- **Sticky Notes:** Present but empty.

---

#### 1.10 Reports Operations

**Overview:**  
Retrieves reports from QuickBooks Online.

**Nodes Involved:**  
- Get a report

**Node Details:**  
Single QuickBooks Tool node for report generation.

- **Potential Failures:** Report type errors, API limits.  
- **Sticky Notes:** Present but empty.

---

#### 1.11 Vendors Operations

**Overview:**  
Manages vendor creation, retrieval (single and many), and updating.

**Nodes Involved:**  
- Create a vendor  
- Get a vendor  
- Get many vendors  
- Update a vendor

**Node Details:**  
QuickBooks Tool nodes for vendor entity.

- **Potential Failures:** Authentication issues, invalid vendor IDs.  
- **Sticky Notes:** Present but empty.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|--------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow Overview 0         | Sticky Note                      | Overview / Documentation        |                               |                               |                                                                                                 |
| QuickBooks Online Tool MCP Server | MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger) | Central entry point for MCP commands | External Webhook               | All QuickBooks Tool nodes      |                                                                                                 |
| Create a bill               | QuickBooks Tool                  | Create bill                    | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Delete a bill               | QuickBooks Tool                  | Delete bill                    | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get a bill                  | QuickBooks Tool                  | Retrieve single bill           | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many bills              | QuickBooks Tool                  | Retrieve multiple bills        | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update a bill               | QuickBooks Tool                  | Update bill                   | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 1               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Create a customer           | QuickBooks Tool                  | Create customer                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get a customer              | QuickBooks Tool                  | Get single customer            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many customers          | QuickBooks Tool                  | Get multiple customers         | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update a customer           | QuickBooks Tool                  | Update customer                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 2               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Create an employee          | QuickBooks Tool                  | Create employee                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get an employee             | QuickBooks Tool                  | Get single employee            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many employees          | QuickBooks Tool                  | Get multiple employees         | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update an employee          | QuickBooks Tool                  | Update employee                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 3               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Create an estimate          | QuickBooks Tool                  | Create estimate                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Delete an estimate          | QuickBooks Tool                  | Delete estimate                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get an estimate             | QuickBooks Tool                  | Get single estimate            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many estimates          | QuickBooks Tool                  | Get multiple estimates         | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Send an estimate            | QuickBooks Tool                  | Send estimate via email        | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update an estimate          | QuickBooks Tool                  | Update estimate                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 4               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Create an invoice           | QuickBooks Tool                  | Create invoice                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Delete an invoice           | QuickBooks Tool                  | Delete invoice                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get an invoice              | QuickBooks Tool                  | Get single invoice            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many invoices           | QuickBooks Tool                  | Get multiple invoices         | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Send an invoice             | QuickBooks Tool                  | Send invoice via email        | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update an invoice           | QuickBooks Tool                  | Update invoice                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Void an invoice             | QuickBooks Tool                  | Void invoice                  | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 5               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Get an item                 | QuickBooks Tool                  | Get single item               | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many items              | QuickBooks Tool                  | Get multiple items            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 6               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Create a payment            | QuickBooks Tool                  | Create payment                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Delete a payment            | QuickBooks Tool                  | Delete payment                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get a payment               | QuickBooks Tool                  | Get single payment            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many payments           | QuickBooks Tool                  | Get multiple payments         | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Send a payment             | QuickBooks Tool                  | Send payment                  | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update a payment            | QuickBooks Tool                  | Update payment                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Void a payment              | QuickBooks Tool                  | Void payment                  | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 7               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Get a purchase              | QuickBooks Tool                  | Get single purchase           | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many purchases          | QuickBooks Tool                  | Get multiple purchases        | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 8               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Get a report                | QuickBooks Tool                  | Get report                   | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 9               | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |
| Create a vendor             | QuickBooks Tool                  | Create vendor                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get a vendor                | QuickBooks Tool                  | Get single vendor            | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Get many vendors            | QuickBooks Tool                  | Get multiple vendors         | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Update a vendor             | QuickBooks Tool                  | Update vendor                | MCP Trigger                   | MCP Trigger                   |                                                                                                 |
| Sticky Note 10              | Sticky Note                     | Documentation                  |                               |                               |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add a new node: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set a unique webhook ID or allow n8n to generate one.  
   - This node will serve as the single entry point for all QuickBooks operations.

2. **Configure QuickBooks OAuth2 Credentials**  
   - In n8n credentials, create or configure QuickBooks OAuth2 credentials with client ID, secret, and tokens.  
   - Make sure the credentials have proper scopes for all operations.

3. **Add QuickBooks Tool Nodes for Each Operation**  
   For each QuickBooks entity and operation listed below, create a `quickbooksTool` node with the following steps:  

   - **Entities and Operations:**  
     - Bills: Create, Delete, Get, Get many, Update  
     - Customers: Create, Get, Get many, Update  
     - Employees: Create, Get, Get many, Update  
     - Estimates: Create, Delete, Get, Get many, Send, Update  
     - Invoices: Create, Delete, Get, Get many, Send, Update, Void  
     - Items: Get, Get many  
     - Payments: Create, Delete, Get, Get many, Send, Update, Void  
     - Purchases: Get, Get many  
     - Reports: Get  
     - Vendors: Create, Get, Get many, Update  

   - **For each node:**  
     - Select the QuickBooks OAuth2 credentials.  
     - Set the operation to match the node name (e.g., "Create a bill" sets operation to create bill).  
     - Leave other parameters dynamic, assuming inputs come from the MCP Trigger payload.  
     - Position nodes logically grouped by entity for clarity.

4. **Connect MCP Trigger Output to Each QuickBooks Tool Node**  
   - Connect the MCP Trigger node’s output to the input of all QuickBooks Tool nodes.  
   - Use the `ai_tool` output port for connections as per node type.

5. **Connect Each QuickBooks Tool Node Back to MCP Trigger**  
   - Connect the output of each QuickBooks Tool node back to the MCP Trigger node input to return results.

6. **Add Sticky Notes for Documentation** (Optional)  
   - Add sticky notes near each entity block for visual grouping or documentation purposes.

7. **Set Workflow Settings**  
   - Set the workflow timezone to America/New_York.  
   - Activate the workflow to start listening on the webhook.

8. **Test the MCP Endpoint**  
   - Send test MCP commands specifying operation and parameters to the webhook URL.  
   - Verify responses and logs for correct operation.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                      |
|------------------------------------------------------------------------------------------------|------------------------------------|
| The workflow uses QuickBooks Online credentials with OAuth2 authentication. Ensure tokens are refreshed regularly. | Credential management in n8n        |
| MCP Trigger node requires n8n version that supports LangChain MCP features.                    | n8n official docs for MCP Trigger   |
| QuickBooks API has rate limits; implement error handling and retries as needed.                | QuickBooks API rate limiting docs   |
| Each QuickBooks Tool node dynamically receives commands and parameters from the MCP trigger.  | Dynamic input handling in n8n       |
| No sticky notes currently contain content; consider adding comments for clarity during maintenance. | Workflow documentation best practice|

---

**Disclaimer:** The provided analysis is based exclusively on the n8n workflow JSON data and adheres strictly to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.