Create a QuickBooks invoice on a new Onfleet Task creation

https://n8nworkflows.xyz/workflows/create-a-quickbooks-invoice-on-a-new-onfleet-task-creation-1546


# Create a QuickBooks invoice on a new Onfleet Task creation

### 1. Workflow Overview

This workflow automates the creation of a QuickBooks Online invoice whenever a new task is created in Onfleet, a last-mile delivery management platform. It is designed to streamline billing processes by connecting Onfleet task events with QuickBooks Online invoicing, enabling businesses to generate invoices automatically for upcoming deliveries.

The workflow consists of two main logical blocks:

- **1.1 Onfleet Event Listener:** Listens for new task creation events in Onfleet.
- **1.2 QuickBooks Invoice Creation:** Creates an invoice in QuickBooks Online based on the received Onfleet task event.

---

### 2. Block-by-Block Analysis

#### 1.1 Onfleet Event Listener

- **Overview:**  
  This block monitors Onfleet for new task creation events. When a task is created, it triggers the workflow to proceed with invoice generation.

- **Nodes Involved:**  
  - Onfleet Trigger

- **Node Details:**

  - **Node Name:** Onfleet Trigger  
  - **Type and Role:** Onfleet Trigger node; serves as an event listener for Onfleet API webhook events.  
  - **Configuration Choices:**  
    - Trigger event set to `taskCreated` (listens specifically for new task creation).  
    - No additional fields configured.  
  - **Key Expressions / Variables:** None by default; captures raw event data from Onfleet.  
  - **Input / Output Connections:**  
    - Input: None (trigger node).  
    - Output: Connected to the QuickBooks Online node.  
  - **Version-Specific Requirements:** Requires n8n version supporting Onfleet Trigger node v1.  
  - **Potential Failure Types:**  
    - Authentication errors if API key is invalid or missing.  
    - Webhook registration issues if Onfleet credentials are misconfigured.  
    - Network timeouts or connectivity issues.  
  - **Sub-workflow Reference:** None.

#### 1.2 QuickBooks Invoice Creation

- **Overview:**  
  This block receives the Onfleet task creation event data and uses it to create a new invoice in QuickBooks Online.

- **Nodes Involved:**  
  - QuickBooks Online

- **Node Details:**

  - **Node Name:** QuickBooks Online  
  - **Type and Role:** QuickBooks node; creates an invoice resource in QuickBooks Online.  
  - **Configuration Choices:**  
    - Resource: `invoice`  
    - Operation: `create`  
    - Additional fields set with empty or zeroed defaults:  
      - Balance: 0  
      - TxnDate: empty (defaults to current date if not set)  
      - ShipAddr: empty  
      - BillEmail: empty  
    - No invoice line items configured (empty Line array), so invoice will have no billed products or services unless modified.  
  - **Key Expressions / Variables:** None configured by default; requires mapping to Onfleet data for practical use.  
  - **Input / Output Connections:**  
    - Input: Receives data from Onfleet Trigger node.  
    - Output: None (end of workflow).  
  - **Version-Specific Requirements:** Requires n8n version supporting QuickBooks node v1.  
  - **Potential Failure Types:**  
    - Authentication errors if QuickBooks credentials are invalid or expired.  
    - API errors if required invoice fields are incomplete or invalid.  
    - Data mapping errors if input data from Onfleet is missing or malformed.  
    - Network or rate limiting issues from QuickBooks API.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name         | Node Type               | Functional Role               | Input Node(s)    | Output Node(s)   | Sticky Note                                                                                      |
|-------------------|-------------------------|------------------------------|------------------|------------------|------------------------------------------------------------------------------------------------|
| Onfleet Trigger   | Onfleet Trigger         | Listen for new Onfleet tasks | None             | QuickBooks Online| Update with your Onfleet credentials. Register for Onfleet API key at https://onfleet.com/signup. See https://support.onfleet.com/hc/en-us/articles/360045763852-Webhooks for webhook info.|
| QuickBooks Online | QuickBooks node         | Create QuickBooks invoice     | Onfleet Trigger  | None             | Update with your QuickBooks Online credentials.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Onfleet Trigger node:**  
   - Type: Onfleet Trigger  
   - Set **Trigger On** to `taskCreated` to listen for new task events.  
   - Configure credentials: Provide your Onfleet API key (register at https://onfleet.com/signup if needed).  
   - No additional fields required unless custom filtering is desired.  
   - Position node as the workflow start.

2. **Create QuickBooks Online node:**  
   - Type: QuickBooks  
   - Set **Resource** to `invoice`.  
   - Set **Operation** to `create`.  
   - Leave **Line** array empty initially or configure as needed to add invoice line items.  
   - In **Additional Fields**, set:  
     - `Balance`: 0 (default).  
     - `TxnDate`: leave empty to default to current date.  
     - `ShipAddr` and `BillEmail`: leave empty or map accordingly.  
   - Configure credentials: Connect your QuickBooks Online OAuth2 credentials with appropriate permissions to create invoices.  
   - Connect the output of the Onfleet Trigger node to this QuickBooks node input.

3. **Connect nodes:**  
   - Link Onfleet Trigger node's main output (index 0) to QuickBooks Online node's main input (index 0).

4. **Activate the workflow:**  
   - Ensure both nodes have valid credentials.  
   - Activate the workflow to listen for new Onfleet tasks and create invoices accordingly.

5. **Optional modifications:**  
   - Add data transformation or function nodes if you want to map Onfleet task data to QuickBooks invoice fields (e.g., customer info, line items).  
   - Handle error workflows with error trigger nodes or additional logic.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                            |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Onfleet API key registration and documentation: https://onfleet.com/signup                           | Onfleet credential setup and onboarding                                                   |
| Onfleet webhooks detailed documentation: https://support.onfleet.com/hc/en-us/articles/360045763852-Webhooks | Understanding Onfleet webhook events and configuration                                    |
| QuickBooks Online API requires OAuth2 credentials with invoice creation permissions                  | QuickBooks credential setup                                                                |
| Consider enriching invoice creation with task details for practical billing (e.g., line items)       | Enhancing workflow functionality                                                           |