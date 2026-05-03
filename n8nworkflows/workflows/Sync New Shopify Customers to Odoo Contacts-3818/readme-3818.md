Sync New Shopify Customers to Odoo Contacts

https://n8nworkflows.xyz/workflows/sync-new-shopify-customers-to-odoo-contacts-3818


# Sync New Shopify Customers to Odoo Contacts

---

### 1. Workflow Overview

This workflow automates the synchronization of new customers created in Shopify with the contacts database in Odoo. Its primary goal is to prevent duplicate contacts by verifying if a customer email from Shopify already exists in Odoo before creating a new contact record.

The workflow is composed of four logical blocks:

- **1.1 Input Reception:** Detects new customer creation events in Shopify.
- **1.2 Data Lookup:** Searches Odoo contacts by email to check for existing entries.
- **1.3 Decision Making:** Evaluates the search results to determine if a new contact should be created.
- **1.4 Data Creation:** Creates a new contact in Odoo using Shopify customer details if no existing contact is found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow each time a new customer is created in Shopify, providing the customer data for downstream processing.

- **Nodes Involved:**  
  - Shopify Trigger

- **Node Details:**  
  - **Shopify Trigger**  
    - Type: Trigger node  
    - Role: Listens for the "customers/create" topic in Shopify via webhook.  
    - Configuration:  
      - Topic: `customers/create` — triggers when a new customer is created.  
      - Authentication: OAuth2 access token via stored Shopify credentials named "Evozard - Shopify".  
      - Webhook ID: Fixed internal webhook identifier for n8n.  
    - Input: None (trigger node).  
    - Output: JSON payload of the newly created Shopify customer, including email, addresses, phone, and name fields.  
    - Edge Cases / Failures:  
      - Authentication failure if Shopify access token expires or is revoked.  
      - Missing or malformed customer data in the webhook payload.  
      - Network or webhook delivery failures.

#### 1.2 Data Lookup

- **Overview:**  
  This block queries the Odoo system to locate any existing contact whose email matches the Shopify customer's email.

- **Nodes Involved:**  
  - Search Odoo Contact

- **Node Details:**  
  - **Search Odoo Contact**  
    - Type: Odoo node (custom resource query)  
    - Role: Retrieves existing Odoo contacts filtered by email address.  
    - Configuration:  
      - Resource: Custom resource "res.partner" (Odoo contact model).  
      - Operation: `getAll` with filter on `email` field equal to the Shopify customer's email.  
      - Limit: 1 — only one match is needed to confirm existence.  
      - Credentials: Odoo API credentials named "Odoo 148.66.157.208:8069".  
      - Always Output Data: true — ensures output even if no match found.  
    - Input: Receives Shopify customer data from trigger node.  
    - Output: JSON array with contact(s) matching the email or empty if none found.  
    - Key Expressions: Filters by `$('Shopify Trigger').item.json.email` dynamically.  
    - Edge Cases / Failures:  
      - Odoo API authentication failures.  
      - Network timeouts or API rate limits.  
      - No contact found (empty array) handled downstream.  
      - Malformed or missing email in Shopify data causing filter failure.

#### 1.3 Decision Making

- **Overview:**  
  This block checks if the Odoo search yielded an existing contact by examining the search output, then makes a decision to either continue creating a new contact or stop the workflow.

- **Nodes Involved:**  
  - Code  
  - Filter

- **Node Details:**  
  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Processes the search results to extract existence flag and customer details for conditional logic.  
    - Configuration:  
      - Runs once per item (runOnceForEachItem).  
      - JavaScript extracts:  
        - `existing` boolean set to true if Odoo contact search returned an ID, false otherwise.  
        - `contact_detail` stores Shopify customer JSON.  
      - Console logs present for debugging.  
    - Input: Odoo search results and Shopify trigger data.  
    - Output: JSON with `existing` (boolean) and `contact_detail` (customer info).  
    - Edge Cases / Failures:  
      - Missing or unexpected structure in Odoo search result causing exception.  
      - Console.log statements may not work in all execution environments.  

  - **Filter**  
    - Type: Filter node  
    - Role: Allows workflow continuation only if no existing contact was found (`existing` is false).  
    - Configuration:  
      - Condition: Boolean check if `existing` equals false.  
      - Uses expression `={{ $json.existing }}` to evaluate existence.  
    - Input: From Code node output.  
    - Output: Passes data to Create Contact node if condition met; otherwise, workflow ends.  
    - Edge Cases / Failures:  
      - Expression evaluation errors if `existing` field missing.  
      - No output path if condition fails (intended behavior).

#### 1.4 Data Creation

- **Overview:**  
  This block creates a new contact record in Odoo using the customer information provided by Shopify, only when no existing email match is found.

- **Nodes Involved:**  
  - Create Contact

- **Node Details:**  
  - **Create Contact**  
    - Type: Odoo node (create operation)  
    - Role: Inserts a new contact in Odoo with detailed customer information.  
    - Configuration:  
      - Resource: Custom resource `res.partner`.  
      - Operation: Create new record.  
      - Fields to set:  
        - `name`: Customer full name extracted from first address name field.  
        - `email`: Shopify customer email.  
        - `street`: Primary address line 1.  
        - `street2`: Address line 2 (optional).  
        - `city`: City name.  
        - `zip`: Postal code.  
        - `phone`: Phone number from the address.  
      - Credentials: Same Odoo API credentials as Search node.  
      - Always Output Data: false (does not pass output further).  
    - Input: Filter node output (only if no existing contact).  
    - Output: None (workflow ends after creation).  
    - Key Expressions: Uses dynamic expressions to extract fields from Shopify trigger data.  
    - Edge Cases / Failures:  
      - Missing address or phone fields in Shopify data may result in incomplete contact creation.  
      - Odoo API errors or validation failures.  
      - Network issues or authentication errors.

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                      | Input Node(s)     | Output Node(s)       | Sticky Note                                                                                  |
|---------------------|-------------------------------|------------------------------------|-------------------|----------------------|----------------------------------------------------------------------------------------------|
| Shopify Trigger      | Shopify Trigger                | Trigger on new Shopify customer    | None              | Search Odoo Contact   |                                                                                              |
| Search Odoo Contact  | Odoo (custom resource query)  | Lookup contact by email in Odoo    | Shopify Trigger   | Code                 |                                                                                              |
| Code                | Code                          | Determine if contact exists        | Search Odoo Contact| Filter               |                                                                                              |
| Filter              | Filter                        | Proceed only if no existing contact| Code              | Create Contact       |                                                                                              |
| Create Contact       | Odoo (create record)           | Create new Odoo contact            | Filter            | None                 |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger Node**  
   - Add a Shopify Trigger node.  
   - Configure the node to listen to the topic: `customers/create`.  
   - Set authentication using OAuth2 credentials for Shopify (named e.g. "Evozard - Shopify").  
   - Save the webhook ID (auto-generated by n8n).  

2. **Add Odoo Search Contact Node**  
   - Add an Odoo node.  
   - Set Resource to `custom` and enter `res.partner` as Custom Resource.  
   - Operation: `getAll`.  
   - Add a filter with one condition:  
     - Field name: `email`  
     - Value: `={{ $('Shopify Trigger').item.json.email }}` (expression referencing Shopify Trigger email).  
   - Limit results to 1.  
   - Select Odoo API credentials (e.g., "Odoo 148.66.157.208:8069").  

3. **Add Code Node**  
   - Add a Code node.  
   - Set mode to `runOnceForEachItem`.  
   - Paste the following JavaScript code:  
     ```javascript
     var contact_detail = $('Shopify Trigger').item.json;
     var existing_contact = $('Search Odoo Contact').item.json;
     return {existing: existing_contact.id ? true : false, contact_detail: contact_detail};
     ```  
   - This extracts a boolean flag `existing` indicating presence of contact and passes customer data.  

4. **Add Filter Node**  
   - Add a Filter node.  
   - Configure the condition:  
     - Boolean condition on field `existing` must be false (`={{ $json.existing }}`).  
   - Connect Code node to Filter node.  
   - If condition is true (no existing contact), pass data to next node. Otherwise, end workflow here.  

5. **Add Odoo Create Contact Node**  
   - Add another Odoo node.  
   - Set Resource to `custom` and Custom Resource to `res.partner`.  
   - Operation: Create record.  
   - Add fields with values set by expressions referencing Shopify Trigger:  
     - `name`: `={{ $('Shopify Trigger').item.json.addresses[0].name }}`  
     - `email`: `={{ $('Shopify Trigger').item.json.email }}`  
     - `street`: `={{ $('Shopify Trigger').item.json.addresses[0].address1 }}`  
     - `street2`: `={{ $('Shopify Trigger').item.json.addresses[0].address2 }}`  
     - `city`: `={{ $('Shopify Trigger').item.json.addresses[0].city }}`  
     - `zip`: `={{ $('Shopify Trigger').item.json.addresses[0].zip }}`  
     - `phone`: `={{ $('Shopify Trigger').item.json.addresses[0].phone }}`  
   - Set Odoo credentials same as previous.  

6. **Connect Nodes in Order**  
   - Shopify Trigger -> Search Odoo Contact -> Code -> Filter -> Create Contact  

7. **Activate Workflow**  
   - Test workflow with new Shopify customers.  
   - Monitor logs for errors or missing data issues.

---

### 5. General Notes & Resources

| Note Content                                                        | Context or Link                                   |
|-------------------------------------------------------------------|--------------------------------------------------|
| Odoo contacts are stored in the `res.partner` model.             | Odoo official documentation for `res.partner`.  |
| Shopify webhook topic used: `customers/create`.                   | Shopify webhook topics documentation.            |
| Ensure Shopify credentials have correct scopes for webhooks.     | Shopify app configuration.                        |
| Odoo API credentials must have create and read permissions.       | Odoo user rights and API setup.                   |
| Handle possible missing address or phone data gracefully.         | Potential data quality improvement area.          |

---

This document provides a comprehensive understanding and stepwise reproduction instructions for the "Sync New Shopify Customers to Odoo Contacts" workflow, ensuring clarity for advanced users and AI-based automation.