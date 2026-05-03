Capture and Structure Web Form Leads for Odoo CRM (v15-v18 Compatible)

https://n8nworkflows.xyz/workflows/capture-and-structure-web-form-leads-for-odoo-crm--v15-v18-compatible--5477


# Capture and Structure Web Form Leads for Odoo CRM (v15-v18 Compatible)

---

### 1. Workflow Overview

This workflow is designed to capture leads submitted via a web form and create corresponding opportunities in an Odoo CRM system compatible with versions 15 to 18. It processes incoming HTTP requests via a webhook, transforms the data as needed, creates an opportunity in Odoo, and then sends a response back to the originator of the request.

The workflow logic is structured into three main blocks:

- **1.1 Input Reception:** Captures incoming web form data through a webhook.
- **1.2 Data Processing:** Transforms or prepares the data for Odoo CRM using a code node.
- **1.3 CRM Integration & Response:** Creates a new opportunity in Odoo and responds to the webhook sender with the operation result.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures the incoming lead data through a webhook HTTP endpoint, providing the entry point for the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Node Name:** Webhook  
    - **Type:** Webhook (n8n-nodes-base.webhook)  
    - **Technical Role:** Entry point that listens for HTTP POST requests containing web form data.  
    - **Configuration:** Uses a unique webhook URL (webhookId: a5555983-beff-4f5e-9ec0-0b7425495df3). No additional parameters configured.  
    - **Key Expressions/Variables:** The incoming HTTP request body is the primary input.  
    - **Input Connections:** None (start node).  
    - **Output Connections:** Passes data to the "Code" node.  
    - **Version Requirements:** Compatible with n8n version supporting webhook node v1.  
    - **Edge Cases / Potential Failures:**  
      - Incoming request missing required data fields.  
      - HTTP method other than POST could result in no trigger.  
      - Network or authentication restrictions if webhook URL is not publicly accessible.  
    - **Sub-workflow Reference:** None

#### 2.2 Data Processing

- **Overview:**  
  This block executes custom JavaScript code to transform or structure the incoming webhook data into a format suitable for Odoo CRM opportunity creation.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Node Name:** Code  
    - **Type:** Code (n8n-nodes-base.code)  
    - **Technical Role:** Processes raw webhook input, applies any necessary data mapping, validation, or enrichment before passing data to Odoo.  
    - **Configuration:** No explicit parameters shown, but likely contains JavaScript code to parse and format input data.  
    - **Key Expressions/Variables:** Uses `items` input from the webhook; outputs transformed items.  
    - **Input Connections:** Receives data from Webhook node.  
    - **Output Connections:** Sends transformed data to "Create Opportunity" node.  
    - **Version Requirements:** Uses Code node version 2, requiring n8n versions supporting Code v2.  
    - **Edge Cases / Potential Failures:**  
      - Syntax errors or runtime exceptions in code.  
      - Unexpected input data formats causing failures.  
      - Empty or incomplete data after processing.  
    - **Sub-workflow Reference:** None

#### 2.3 CRM Integration & Response

- **Overview:**  
  This block creates a new opportunity record in Odoo CRM using the processed data and then responds to the original webhook request to confirm success or failure.

- **Nodes Involved:**  
  - Create Opportunity  
  - Respond to Webhook

- **Node Details:**

  - **Node Name:** Create Opportunity  
    - **Type:** Odoo (n8n-nodes-base.odoo)  
    - **Technical Role:** Sends a request to Odoo CRM to create a new opportunity record with the structured lead data.  
    - **Configuration:** Parameters would include Odoo instance credentials, model set to "crm.lead" or equivalent, and field mappings for opportunity creation. Specific configuration is not detailed but implied.  
    - **Key Expressions/Variables:** Receives input data from Code node; outputs result of creation.  
    - **Input Connections:** From Code node.  
    - **Output Connections:** To Respond to Webhook node.  
    - **Version Requirements:** Compatible with Odoo versions 15 to 18, requires valid Odoo API credentials.  
    - **Edge Cases / Potential Failures:**  
      - Authentication failure with Odoo API.  
      - API errors due to invalid data or connectivity issues.  
      - Version-specific API changes affecting request payload or endpoints.  
    - **Sub-workflow Reference:** None

  - **Node Name:** Respond to Webhook  
    - **Type:** Respond to Webhook (n8n-nodes-base.respondToWebhook)  
    - **Technical Role:** Sends an HTTP response back to the original webhook caller, indicating success or failure of opportunity creation.  
    - **Configuration:** No specific parameters shown but generally includes status code and response body configuration.  
    - **Key Expressions/Variables:** Uses output from Create Opportunity node to formulate response.  
    - **Input Connections:** From Create Opportunity node.  
    - **Output Connections:** None (end node).  
    - **Version Requirements:** Node version 1.1 used; compatible with n8n versions supporting this.  
    - **Edge Cases / Potential Failures:**  
      - Failure to respond due to workflow errors upstream.  
      - Timeout if Create Opportunity node is slow.  
    - **Sub-workflow Reference:** None

---

### 3. Summary Table

| Node Name         | Node Type                | Functional Role               | Input Node(s)       | Output Node(s)     | Sticky Note                 |
|-------------------|--------------------------|------------------------------|---------------------|--------------------|-----------------------------|
| Webhook           | Webhook                  | Capture incoming web leads    | -                   | Code               |                             |
| Code              | Code                     | Transform input data          | Webhook             | Create Opportunity  |                             |
| Create Opportunity| Odoo                     | Create opportunity in Odoo CRM| Code                | Respond to Webhook  |                             |
| Respond to Webhook| Respond to Webhook       | Send HTTP response            | Create Opportunity  | -                  |                             |
| Sticky Note       | Sticky Note              | -                            | -                   | -                  |                             |
| Sticky Note1      | Sticky Note              | -                            | -                   | -                  |                             |
| Sticky Note2      | Sticky Note              | -                            | -                   | -                  |                             |

*No sticky notes content was provided in the JSON.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node**  
   - Add a new node of type **Webhook**.  
   - Configure it to listen for HTTP POST requests.  
   - Copy or assign a unique webhook URL (this will be the endpoint to which web form submissions are sent).  
   - Save the node.

2. **Add a Code Node**  
   - Add a **Code** node after the Webhook node.  
   - Connect the Webhook node's output to the Code node's input.  
   - In the Code node, write JavaScript to:  
     - Parse and validate the incoming webhook data.  
     - Map the data fields to the structure expected by Odoo CRM opportunity records.  
     - Handle missing fields or data normalization.  
   - Use `items` as input and return transformed `items`.  
   - Save the node.

3. **Add an Odoo Node**  
   - Add an **Odoo** node after the Code node.  
   - Connect the Code node's output to the Odoo node's input.  
   - Configure the Odoo node with:  
     - Credentials for your Odoo instance (API URL, database, username, password/API key).  
     - Model set to `crm.lead` or equivalent opportunity model.  
     - Operation set to `Create` to create a new record.  
     - Map the transformed data fields from the Code node to the Odoo opportunity fields (e.g., name, email, phone, description).  
   - Save the node.

4. **Add a Respond to Webhook Node**  
   - Add a **Respond to Webhook** node after the Odoo node.  
   - Connect the Odoo node's output to this node.  
   - Configure it to send a JSON response indicating success or error status back to the HTTP caller.  
   - Optionally include the created opportunity ID or error message in the response body.  
   - Save the node.

5. **Configure Workflow Execution Order**  
   - Ensure the nodes are connected in this order:  
     Webhook → Code → Create Opportunity (Odoo) → Respond to Webhook

6. **Activate the Workflow**  
   - Save and activate the workflow to start receiving web form leads and creating Odoo opportunities automatically.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                        |
|------------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow is compatible with Odoo CRM versions 15 to 18 inclusive.       | Workflow Title and Description                         |
| No sticky notes content was provided; consider documenting custom code logic.| Workflow JSON analysis                                |
| For configuring Odoo credentials, refer to official n8n Odoo node docs:      | https://docs.n8n.io/integrations/nodes/n8n-nodes-base.odoo/ |
| Ensure webhook URL is secured if exposed publicly to avoid spam submissions. | Security best practice                                 |

---

Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---