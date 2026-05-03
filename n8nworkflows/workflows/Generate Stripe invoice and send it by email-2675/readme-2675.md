Generate Stripe invoice and send it by email

https://n8nworkflows.xyz/workflows/generate-stripe-invoice-and-send-it-by-email-2675


# Generate Stripe invoice and send it by email

### 1. Workflow Overview

This workflow automates the process of generating and sending Stripe invoices via email by orchestrating the necessary API calls in the correct sequence. It is designed for users who want to create Stripe invoices programmatically and ensure customers receive them without manual intervention.

The workflow logically breaks down into four main blocks:

- **1.1 Input Trigger**: Manual initiation of the workflow.
- **1.2 Customer Creation**: Creating a new customer in Stripe.
- **1.3 Invoice Item Specification**: Adding invoice items to the created customer.
- **1.4 Invoice Generation and Finalization**: Creating the invoice and finalizing it so that Stripe sends it automatically to the customer’s email.

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

**Overview:**  
This block starts the workflow manually to test or trigger invoice generation on demand.

**Nodes Involved:**  
- When clicking ‘Test workflow’

**Node Details:**  
- **Type and Technical Role:** Manual Trigger node; initiates the workflow execution without external input.  
- **Configuration Choices:** Default manual trigger configuration; no parameters needed.  
- **Key Expressions/Variables:** None.  
- **Input/Output Connections:** Output connects to "Create Customer" node.  
- **Version-specific Requirements:** None.  
- **Potential Failure Modes:** None (manual trigger).  
- **Sub-workflow Reference:** None.

#### 2.2 Customer Creation

**Overview:**  
Creates a customer record in Stripe, which is required as a prerequisite for adding invoice items and generating invoices.

**Nodes Involved:**  
- Create Customer

**Node Details:**  
- **Type and Technical Role:** Stripe node configured to create a customer via Stripe API.  
- **Configuration Choices:** Uses Stripe credentials set in n8n; creates a new customer with default or predefined data (customer details not explicitly specified in the JSON, likely using defaults or parameters).  
- **Key Expressions/Variables:** None explicitly defined; typically would include customer email, name, etc.  
- **Input/Output Connections:** Receives input from the manual trigger; outputs to "Stripe | Invoice Items".  
- **Version-specific Requirements:** Stripe API credentials must be configured in n8n.  
- **Potential Failure Modes:** Authentication errors, invalid customer data, Stripe API rate limits, network timeouts.  
- **Sub-workflow Reference:** None.

#### 2.3 Invoice Item Specification

**Overview:**  
Specifies the invoice items for the customer by making an HTTP request to Stripe’s API to add line items to the invoice.

**Nodes Involved:**  
- Stripe | Invoice Items

**Node Details:**  
- **Type and Technical Role:** HTTP Request node; performs a POST (or appropriate method) call to Stripe API to add invoice items.  
- **Configuration Choices:** Custom HTTP request configured with Stripe API endpoint for invoice items; uses Stripe API key for authentication via headers.  
- **Key Expressions/Variables:** Likely uses output from "Create Customer" to get the customer ID dynamically in the request body or URL.  
- **Input/Output Connections:** Input from "Create Customer"; output to "Stripe | Create invoice".  
- **Version-specific Requirements:** Requires Stripe API version compatibility with invoice item endpoints; HTTP Request node version 4.2.  
- **Potential Failure Modes:** Invalid customer ID, malformed request body, authentication errors, Stripe API changes, network issues.  
- **Sub-workflow Reference:** None.

#### 2.4 Invoice Generation and Finalization

**Overview:**  
This block creates the invoice in Stripe and then finalizes it, triggering Stripe to send the invoice email to the customer.

**Nodes Involved:**  
- Stripe | Create invoice  
- Stripe | Finalize invoice

**Node Details:**  

- **Stripe | Create invoice**  
  - **Type and Technical Role:** HTTP Request node; sends a request to Stripe to create an invoice for the specified customer and invoice items.  
  - **Configuration Choices:** Configured with the Stripe API endpoint for invoice creation, using the customer and invoice item information from previous nodes.  
  - **Key Expressions/Variables:** Uses data from "Stripe | Invoice Items" output to link invoice items and customer ID.  
  - **Input/Output Connections:** Input from "Stripe | Invoice Items"; output to "Stripe | Finalize invoice".  
  - **Version-specific Requirements:** Stripe API version compatibility; HTTP Request node version 4.2.  
  - **Potential Failure Modes:** Invalid invoice data, API authentication failure, network errors, missing invoice items.  
  - **Sub-workflow Reference:** None.

- **Stripe | Finalize invoice**  
  - **Type and Technical Role:** HTTP Request node; sends a request to finalize the created invoice, which triggers Stripe to send it to the customer’s email.  
  - **Configuration Choices:** Uses Stripe API endpoint to finalize the invoice by invoice ID obtained from the previous node.  
  - **Key Expressions/Variables:** Invoice ID from "Stripe | Create invoice" output.  
  - **Input/Output Connections:** Input from "Stripe | Create invoice". No further output connected.  
  - **Version-specific Requirements:** Stripe API version must support invoice finalization.  
  - **Potential Failure Modes:** Invoice not found, invoice already finalized, API authentication errors, network issues.  
  - **Sub-workflow Reference:** None.

### 3. Summary Table

| Node Name             | Node Type             | Functional Role                       | Input Node(s)             | Output Node(s)               | Sticky Note                                         |
|-----------------------|-----------------------|------------------------------------|---------------------------|-----------------------------|----------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger        | Starts workflow execution manually |                           | Create Customer             |                                                    |
| Create Customer        | Stripe Node           | Creates Stripe customer             | When clicking ‘Test workflow’ | Stripe | Invoice Items       |                                                    |
| Stripe | Invoice Items | HTTP Request          | Adds invoice items to customer      | Create Customer            | Stripe | Create invoice        |                                                    |
| Stripe | Create invoice | HTTP Request          | Creates invoice for customer        | Stripe | Invoice Items          | Stripe | Finalize invoice       |                                                    |
| Stripe | Finalize invoice | HTTP Request          | Finalizes invoice and triggers email | Stripe | Create invoice          |                             |                                                    |
| Sticky Note3          | Sticky Note           |                                    |                           |                             |                                                    |
| Sticky Note           | Sticky Note           |                                    |                           |                             |                                                    |
| Sticky Note1          | Sticky Note           |                                    |                           |                             |                                                    |
| Sticky Note2          | Sticky Note           |                                    |                           |                             |                                                    |

*Note: Sticky notes are present in the workflow but contain no textual content in the JSON provided.*

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’". This node initiates the workflow manually.

2. **Create Stripe Customer Node**  
   - Add a **Stripe** node named "Create Customer".  
   - Configure credentials with your Stripe API key in n8n.  
   - Set the operation to **Create Customer**.  
   - Optionally configure customer details to be created (email, name, metadata), or leave default if you want to pass parameters later.  
   - Connect output of "When clicking ‘Test workflow’" to input of "Create Customer".

3. **Create HTTP Request Node for Invoice Items**  
   - Add an **HTTP Request** node named "Stripe | Invoice Items".  
   - Configure the node as follows:  
     - HTTP Method: POST  
     - URL: `https://api.stripe.com/v1/invoiceitems`  
     - Authentication: Bearer Token (use Stripe API key)  
     - Request Body (Form or JSON): Include parameters such as `customer` (use expression to get customer ID from "Create Customer" output), `amount`, `currency`, `description`, etc.  
   - Connect output of "Create Customer" to input of "Stripe | Invoice Items".

4. **Create HTTP Request Node for Invoice Creation**  
   - Add an **HTTP Request** node named "Stripe | Create invoice".  
   - Configure the node as follows:  
     - HTTP Method: POST  
     - URL: `https://api.stripe.com/v1/invoices`  
     - Authentication: Bearer Token with Stripe API key  
     - Request Body: Include `customer` parameter using the customer ID from the "Create Customer" output.  
   - Connect output of "Stripe | Invoice Items" to input of "Stripe | Create invoice".

5. **Create HTTP Request Node for Invoice Finalization**  
   - Add an **HTTP Request** node named "Stripe | Finalize invoice".  
   - Configure the node as follows:  
     - HTTP Method: POST  
     - URL: `https://api.stripe.com/v1/invoices/{{ $json["id"] }}/finalize` (use expression to insert the invoice ID from "Stripe | Create invoice" output)  
     - Authentication: Bearer Token with Stripe API key  
   - Connect output of "Stripe | Create invoice" to input of "Stripe | Finalize invoice".

6. **Test the Workflow**  
   - Run the workflow manually via the "When clicking ‘Test workflow’" trigger.  
   - Ensure your Stripe credentials are correctly configured and that you have the necessary permissions.  
   - Verify that customer creation, invoice item addition, invoice creation, and finalization steps complete successfully.  
   - Verify that Stripe sends the invoice email to the customer.

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                         |
|-----------------------------------------------------------------------------------------------|---------------------------------------|
| Generating Stripe invoices requires multiple API steps: creating customer, invoice items, invoice creation, and finalization.| Workflow purpose overview              |
| Stripe only sends invoice emails after the invoice is finalized via API.                      | Stripe API behavior                    |
| Template creator’s other workflows available at: https://n8n.io/creators/solomon/             | Creator’s n8n workflow templates page |

---

This documentation provides a clear, stepwise understanding and reproduction guide for the "Generate Stripe invoice and send it by email" workflow, allowing advanced users and AI agents to operate, modify, or extend it safely and effectively.