Stripe Payment Order Sync – Auto Retrieve Customer & Product Purchased

https://n8nworkflows.xyz/workflows/stripe-payment-order-sync---auto-retrieve-customer---product-purchased-3391


# Stripe Payment Order Sync – Auto Retrieve Customer & Product Purchased

### 1. Workflow Overview

This workflow automates the synchronization of Stripe payment orders by triggering on successful payment events. Upon receiving a Stripe checkout session completion event, it retrieves detailed session information, including customer and purchased product data, and filters this data to output only the essential fields: customer name, customer email, and product purchased. This streamlined data can then be used for CRM updates, inventory management, notifications, or other post-payment processes.

The workflow is logically divided into three main blocks:

- **1.1 Stripe Event Trigger:** Listens for Stripe checkout session completion events.
- **1.2 Session Data Retrieval:** Fetches detailed session information from Stripe’s API, expanding line items.
- **1.3 Data Filtering and Mapping:** Extracts and formats key customer and product details for downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Stripe Event Trigger

- **Overview:**  
  This block initiates the workflow by listening for Stripe webhook events specifically for completed checkout sessions. It ensures the workflow activates only when a payment is successfully processed.

- **Nodes Involved:**  
  - Stripe Trigger on Payment Event

- **Node Details:**

  - **Node Name:** Stripe Trigger on Payment Event  
  - **Type:** Stripe Trigger (Webhook)  
  - **Role:** Listens for Stripe webhook events, specifically `checkout.session.completed`.  
  - **Configuration:**  
    - Event subscribed: `checkout.session.completed`  
    - Uses Stripe API credentials (test mode in this case).  
    - Webhook ID is preconfigured to receive Stripe events.  
  - **Expressions/Variables:**  
    - Event data accessed via `$json.data.object.id` for session ID.  
  - **Input/Output:**  
    - Input: External Stripe webhook event.  
    - Output: JSON payload containing event data, including the checkout session ID.  
  - **Version Requirements:**  
    - Requires Stripe API credentials configured in n8n.  
    - Webhook must be publicly accessible and registered in Stripe dashboard.  
  - **Potential Failures:**  
    - Webhook misconfiguration or network issues causing missed events.  
    - Invalid or expired Stripe credentials leading to authentication errors.  
    - Event filtering misconfiguration causing no triggers.  
  - **Sub-workflow:** None.

---

#### 2.2 Session Data Retrieval

- **Overview:**  
  This block retrieves detailed checkout session information from Stripe’s API using the session ID obtained from the trigger. It expands the `line_items` to include product details in the response.

- **Nodes Involved:**  
  - Extract Session Information

- **Node Details:**

  - **Node Name:** Extract Session Information  
  - **Type:** HTTP Request  
  - **Role:** Calls Stripe API to fetch full checkout session details including expanded line items.  
  - **Configuration:**  
    - HTTP Method: GET (default for HTTP Request node).  
    - URL dynamically constructed as `https://api.stripe.com/v1/checkout/sessions/{{ $json.data.object.id }}`.  
    - Query parameter: `expand[]=line_items` to include product line items in the response.  
    - Authentication: Uses predefined Stripe API credentials.  
  - **Expressions/Variables:**  
    - Uses expression to inject session ID from trigger node: `{{ $json.data.object.id }}`.  
  - **Input/Output:**  
    - Input: JSON from Stripe Trigger node containing session ID.  
    - Output: JSON with full session details including customer info and line items.  
  - **Version Requirements:**  
    - Requires Stripe API credentials with read access.  
    - HTTP Request node version 4.2 or higher recommended for query parameter handling.  
  - **Potential Failures:**  
    - Invalid session ID causing 404 errors.  
    - API rate limits or network timeouts.  
    - Authentication failures due to invalid or expired credentials.  
  - **Sub-workflow:** None.

---

#### 2.3 Data Filtering and Mapping

- **Overview:**  
  This block extracts and formats the relevant customer and product information from the detailed session data, preparing it for downstream use or integration.

- **Nodes Involved:**  
  - Filter Information

- **Node Details:**

  - **Node Name:** Filter Information  
  - **Type:** Set  
  - **Role:** Maps and filters the session JSON to output only customer name, customer email, and product purchased description.  
  - **Configuration:**  
    - Assigns three new fields:  
      - `Customer Name` from `customer_details.name`  
      - `Customer Email` from `customer_details.email`  
      - `Product Purchased` from `line_items.data[0].description` (first product line item)  
  - **Expressions/Variables:**  
    - Uses expressions such as `={{ $json.customer_details.name }}` to extract data.  
  - **Input/Output:**  
    - Input: Full session JSON from the HTTP Request node.  
    - Output: Simplified JSON with only the three mapped fields.  
  - **Version Requirements:**  
    - Set node version 3.4 or higher recommended for expression support.  
  - **Potential Failures:**  
    - Missing or null fields if the session data is incomplete.  
    - Index errors if `line_items.data` is empty or undefined.  
    - Expression evaluation errors if JSON structure changes.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                     | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|-----------------------------|---------------------|-----------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| Stripe Trigger on Payment Event | Stripe Trigger      | Listens for Stripe payment events | External Stripe webhook       | Extract Session Information  | Ensure Stripe credentials are connected and webhook is configured in Stripe dashboard.       |
| Extract Session Information | HTTP Request        | Retrieves full checkout session data | Stripe Trigger on Payment Event | Filter Information           | Uses Stripe API to expand line items; requires valid session ID from trigger.                 |
| Filter Information          | Set                 | Filters and maps customer/product data | Extract Session Information   | None                        | Maps customer name, email, and first product description; adjust if multiple products exist.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Stripe Trigger Node**  
   - Add a **Stripe Trigger** node named `Stripe Trigger on Payment Event`.  
   - Configure credentials with your Stripe API keys (test or live).  
   - Set event subscription to `checkout.session.completed`.  
   - Ensure the webhook URL generated is registered in your Stripe dashboard under webhook endpoints.  

2. **Create HTTP Request Node**  
   - Add an **HTTP Request** node named `Extract Session Information`.  
   - Set HTTP Method to GET (default).  
   - Set URL to:  
     ```
     https://api.stripe.com/v1/checkout/sessions/{{ $json.data.object.id }}
     ```  
   - Under Query Parameters, add:  
     - Name: `expand[]`  
     - Value: `line_items`  
   - Set Authentication to use the same Stripe API credentials as the trigger node.  
   - Connect the output of `Stripe Trigger on Payment Event` to this node’s input.  

3. **Create Set Node**  
   - Add a **Set** node named `Filter Information`.  
   - In the node parameters, add three fields with the following values:  
     - `Customer Name` = `={{ $json.customer_details.name }}`  
     - `Customer Email` = `={{ $json.customer_details.email }}`  
     - `Product Purchased` = `={{ $json.line_items.data[0].description }}`  
   - Connect the output of `Extract Session Information` to this node’s input.  

4. **Activate and Test**  
   - Activate the workflow.  
   - Trigger a test payment in Stripe to generate a `checkout.session.completed` event.  
   - Verify that the workflow triggers and outputs the filtered customer and product data.  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Ensure your Stripe webhook endpoint URL is publicly accessible and correctly configured in Stripe dashboard. | Stripe Webhook Setup: https://stripe.com/docs/webhooks/setup                                  |
| Customize the Set node to include additional fields if your use case requires more data.         | n8n Set Node Documentation: https://docs.n8n.io/nodes/n8n-nodes-base.set/                      |
| This workflow can be extended to trigger emails, CRM updates, or inventory management actions.   | Consider integrating with Email, CRM, or Database nodes in n8n for full automation.            |
| Use Stripe test mode credentials for initial testing to avoid live transaction impacts.           | Stripe Testing Guide: https://stripe.com/docs/testing                                          |

---

This document provides a complete and detailed reference for understanding, reproducing, and extending the Stripe Payment Order Sync workflow in n8n.