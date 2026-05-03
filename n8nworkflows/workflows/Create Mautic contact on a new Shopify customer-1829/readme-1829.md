Create Mautic contact on a new Shopify customer

https://n8nworkflows.xyz/workflows/create-mautic-contact-on-a-new-shopify-customer-1829


# Create Mautic contact on a new Shopify customer

### 1. Workflow Overview

This workflow automates the creation of a new contact in Mautic whenever a new customer is registered in a Shopify store. It is designed for e-commerce businesses that use Shopify and Mautic for marketing automation and want seamless synchronization of customer data between these platforms.

**Logical Blocks:**

- **1.1 Shopify Trigger:** Listens for new customer creation events in Shopify.
- **1.2 Mautic Contact Creation:** Uses the customer data from Shopify to create a corresponding contact in Mautic.
- **1.3 User Guidance:** Provides a note on how to extend the data sent to Mautic.

---

### 2. Block-by-Block Analysis

#### 1.1 Shopify Trigger

- **Overview:**  
  This block detects when a new customer is created in the Shopify store and initiates the workflow with the customer’s data.

- **Nodes Involved:**  
  - On new customer

- **Node Details:**  
  - **Node Name:** On new customer  
  - **Type:** Shopify Trigger (n8n-nodes-base.shopifyTrigger)  
  - **Technical Role:** Webhook-based trigger node that listens for new customer creation events from Shopify.  
  - **Configuration:**  
    - Topic set to "customers/create" to trigger on new customer creation.  
    - Authentication uses Shopify OAuth Access Token credentials (configured externally).  
  - **Key Expressions/Variables:**  
    - Outputs JSON containing the newly created customer's data, including `email`, `first_name`, and `last_name`.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connects to the "Create contact" node.  
  - **Version-Specific Requirements:** Version 1 node; ensure Shopify API version compatibility.  
  - **Potential Failure Types:**  
    - Authentication errors if Shopify credentials are invalid or expired.  
    - Webhook registration failure if Shopify app permissions are insufficient.  
    - Network or timeout errors.  
  - **Sub-Workflow Reference:** None.

#### 1.2 Mautic Contact Creation

- **Overview:**  
  This block receives customer data from Shopify and creates a new contact in Mautic with that information.

- **Nodes Involved:**  
  - Create contact

- **Node Details:**  
  - **Node Name:** Create contact  
  - **Type:** Mautic node (n8n-nodes-base.mautic)  
  - **Technical Role:** Sends data to the Mautic API to create or update a contact.  
  - **Configuration:**  
    - Email field mapped from Shopify customer’s `email`.  
    - First and last names mapped respectively from `first_name` and `last_name` fields.  
    - Additional fields are empty by default but can be added as needed.  
    - Credentials configured with Mautic API credentials.  
  - **Key Expressions/Variables:**  
    - `email`: `{{$node["On new customer"].json["email"]}}`  
    - `firstName`: `{{$node["On new customer"].json["first_name"]}}`  
    - `lastName`: `{{$node["On new customer"].json["last_name"]}}`  
  - **Input Connections:** Receives input from "On new customer".  
  - **Output Connections:** None (end node).  
  - **Version-Specific Requirements:** Version 1 node; ensure Mautic API compatibility.  
  - **Potential Failure Types:**  
    - Authentication errors if Mautic credentials are invalid or expired.  
    - API errors if required fields are missing or invalid (e.g., invalid email format).  
    - Network or timeout errors.  
  - **Sub-Workflow Reference:** None.

#### 1.3 User Guidance

- **Overview:**  
  This block contains a sticky note providing instructions to extend the workflow by adding more fields to the Mautic contact.

- **Nodes Involved:**  
  - Note

- **Node Details:**  
  - **Node Name:** Note  
  - **Type:** Sticky Note (n8n-nodes-base.stickyNote)  
  - **Technical Role:** Documentation within the workflow UI to guide users.  
  - **Configuration:**  
    - Content:  
      ```
      ### Add more fields to Mautic
      By default, the first name, last name and email are pushed to Mautic. If you require more fields, add it in the `Create contact` node.
      ```  
  - **Input Connections:** None.  
  - **Output Connections:** None.  
  - **Version-Specific Requirements:** None.  
  - **Potential Failure Types:** None.

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role              | Input Node(s)     | Output Node(s) | Sticky Note                                                                                      |
|-----------------|---------------------|-----------------------------|-------------------|----------------|-------------------------------------------------------------------------------------------------|
| On new customer | Shopify Trigger     | Trigger on new Shopify customer creation | None              | Create contact |                                                                                                 |
| Create contact  | Mautic              | Create contact in Mautic using Shopify data | On new customer   | None           |                                                                                                 |
| Note            | Sticky Note         | User guidance on extending fields sent to Mautic | None              | None           | ### Add more fields to Mautic<br>By default, the first name, last name and email are pushed to Mautic. If you require more fields, add it in the `Create contact` node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Shopify Trigger Node:**
   - Add a new node of type **Shopify Trigger**.
   - Set the **Trigger Topic** to `customers/create` to trigger on new customer creation.
   - Configure **Authentication** using Shopify Access Token credentials:
     - Create or select Shopify credentials with the necessary access token.
   - Position this node as the starting point of the workflow.

2. **Create the Mautic Node to Create Contact:**
   - Add a new node of type **Mautic**.
   - Set the operation to create a contact.
   - Map the following fields using expressions referencing the Shopify Trigger node:
     - **Email:** `{{$node["On new customer"].json["email"]}}`
     - **First Name:** `{{$node["On new customer"].json["first_name"]}}`
     - **Last Name:** `{{$node["On new customer"].json["last_name"]}}`
   - Leave additional fields empty by default or add custom fields as needed.
   - Configure **Authentication** using Mautic API credentials:
     - Create or select Mautic API credentials with proper API access.
   - Connect the Shopify Trigger node output to this node’s input.

3. **Add a Sticky Note for User Instructions:**
   - Add a **Sticky Note** node.
   - Enter the content:
     ```
     ### Add more fields to Mautic
     By default, the first name, last name and email are pushed to Mautic. If you require more fields, add it in the `Create contact` node.
     ```
   - Position it appropriately for user visibility.

4. **Review and Activate the Workflow:**
   - Check all credential configurations are valid and test connectivity.
   - Save and activate the workflow.
   - Confirm that when a new customer is created in Shopify, a corresponding contact appears in Mautic with mapped fields.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Shopify credentials setup documentation and requirements.                                          | https://docs.n8n.io/integrations/builtin/credentials/shopify/                                              |
| Mautic credentials setup documentation and requirements.                                           | https://docs.n8n.io/integrations/builtin/credentials/mautic/                                               |
| Custom fields can be added in the Mautic "Create contact" node by extending the `additionalFields`. | See sticky note content within the workflow UI.                                                            |
| Ensure Shopify app permissions include webhook registration for customer creation events.          | Shopify app configuration and permissions guide.                                                           |
| Verify API versions compatibility between n8n nodes and Shopify/Mautic APIs for smooth operation. | Refer to respective API changelogs and n8n documentation for node updates.                                 |

---

This document provides a complete and explicit reference to understand, reproduce, and extend the "Create Mautic contact on a new Shopify customer" workflow in n8n.