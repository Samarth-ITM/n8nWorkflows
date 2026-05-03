Replicate Line Items on New Deal in HubSpot and notify with Slack

https://n8nworkflows.xyz/workflows/replicate-line-items-on-new-deal-in-hubspot-and-notify-with-slack-2367


# Replicate Line Items on New Deal in HubSpot and notify with Slack

### 1. Workflow Overview

This workflow automates the replication of line items from a "won" deal to a newly created deal within HubSpot, eliminating the need for manual copying and reducing errors. It is triggered via a webhook that receives deal IDs, then systematically retrieves associated line items and their product SKUs, fetches detailed product information, creates equivalent line items linked to the new deal, and finally sends a Slack notification confirming success.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts incoming webhook requests containing deal IDs from a HubSpot workflow.
- **1.2 Deal ID Extraction:** Parses and sets the won deal and newly created deal IDs from webhook query parameters.
- **1.3 Line Item Retrieval and Processing:** Fetches line items for the won deal, extracts SKUs, retrieves product details by SKU, and creates new line items associated with the new deal.
- **1.4 Notification:** Sends a Slack message confirming the successful replication of line items.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives the initial trigger from HubSpot via webhook, containing query parameters for deal IDs. This is the entry point to the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Waits for HTTP GET requests at a specified endpoint; initiates workflow execution.  
    - Configuration:  
      - Path set to unique webhook ID (`833df60e-a78f-4a59-8244-9694f27cf8ae`).  
      - No authentication or additional options configured.  
    - Inputs: None (trigger node)  
    - Outputs: Passes incoming HTTP query parameters (`deal_id_won` and `deal_id_created`) as JSON to next node.  
    - Edge cases: Missing or malformed query parameters will disrupt downstream processing.  
    - Version requirements: None specific.  

#### 2.2 Deal ID Extraction

- **Overview:**  
Extracts and sets the deal IDs from the incoming webhook query parameters into workflow variables for subsequent use.

- **Nodes Involved:**  
  - Retrieve deals Ids (Set node)

- **Node Details:**

  - **Retrieve deals Ids**  
    - Type: Set node (data transformation)  
    - Role: Parses query parameters to extract `deal_id_won` and `deal_id_created`.  
    - Configuration:  
      - Assigns `deal_id_won` directly from the webhook query parameter `deal_id_won`.  
      - Extracts `deal_id_created` by applying a regex match to `deal_id_created` query parameter to capture numeric ID (pattern `/0-3-(\d+)$/`), taking the first group.  
    - Expressions used:  
      - `={{ $json.query.deal_id_won }}` for won deal ID.  
      - `={{ $json.query.deal_id_created.match(/0-3-(\d+)$/)[1] }}` for created deal ID extraction.  
    - Inputs: Webhook output JSON.  
    - Outputs: JSON with two fields: `deal_id_won`, `deal_id_created`.  
    - Edge cases: If `deal_id_created` does not match the regex, this will cause errors or undefined values. Input validation or error handling recommended.  
    - Version: 3.4  

#### 2.3 Line Item Retrieval and Processing

- **Overview:**  
Fetches line items associated with the won deal, extracts SKUs, retrieves detailed product information, then creates new line items linked to the newly created deal.

- **Nodes Involved:**  
  - Get deal won line items (HTTP Request)  
  - Get batch SKUs from line items (HTTP Request)  
  - Get Batch Product IDs by SKUs (HTTP Request)  
  - Create Batch line items based on productId and associate to deals (HTTP Request)

- **Node Details:**

  - **Get deal won line items**  
    - Type: HTTP Request (POST)  
    - Role: Calls HubSpot API to retrieve line items linked to the won deal.  
    - Configuration:  
      - URL: `https://api.hubapi.com/crm/v4/associations/deals/line_items/batch/read`  
      - Body JSON: `{ "inputs": [{ "id": "{{ $json.deal_id_won }}" }] }` to batch-read associations for the won deal ID.  
      - Authentication: HubSpot App Token (predefined credential). OAuth2 fallback configured but not primary.  
      - Sends body as JSON.  
    - Inputs: JSON containing `deal_id_won`.  
    - Outputs: JSON results with line item association IDs for the won deal.  
    - Edge cases: API rate limits, invalid deal ID, or auth failures. Ensure token validity.  
    - Version: 4.2  

  - **Get batch SKUs from line items**  
    - Type: HTTP Request (POST)  
    - Role: Retrieves detailed line item data including SKUs by batch reading line item IDs obtained previously.  
    - Configuration:  
      - URL: `https://api.hubapi.com/crm/v3/objects/line_items/batch/read`  
      - Body JSON dynamically constructed with:  
        - `idProperty` = `"hs_object_id"`  
        - `inputs` = Extracts line item IDs via JMESPath: `$jmespath($json.results,`[].to[].{id: to_string(toObjectId)}`)`  
        - `properties`: requests `hs_object_id`, `name`, and `hs_sku` fields.  
      - Query param: `archived=false` to exclude archived line items.  
      - Authentication: HubSpot App Token.  
    - Inputs: Output from previous node with line item associations.  
    - Outputs: Detailed line item objects including SKUs.  
    - Edge cases: Empty line items, missing SKUs, API failures.  
    - Version: 4.2  

  - **Get Batch Product IDs by SKUs**  
    - Type: HTTP Request (POST)  
    - Role: Retrieves product details corresponding to SKUs extracted from line items.  
    - Configuration:  
      - URL: `https://api.hubapi.com/crm/v3/objects/products/batch/read`  
      - Body JSON built with:  
        - `idProperty` = `"hs_sku"`  
        - `inputs` = Extracts SKUs from previous node via JMESPath: `$jmespath($json.results,"[].properties.{id:to_string(hs_sku)}")`  
        - `properties`: requests `idProperty`, `name`, `hs_object_id`, `recurringbillingfrequency`, and `hs_price_eur`.  
      - Authentication: HubSpot App Token.  
    - Inputs: Line items with SKUs.  
    - Outputs: Product details matching SKUs.  
    - Edge cases: Missing or invalid SKUs, products not found, API errors.  
    - Version: 4.2  

  - **Create Batch line items based on productId and associate to deals**  
    - Type: HTTP Request (POST)  
    - Role: Creates new line items for the newly created deal, linking each product by ID and setting quantity to 1.  
    - Configuration:  
      - URL: `https://api.hubapi.com/crm/v3/objects/line_items/batch/create`  
      - Body JSON dynamically maps product IDs to new line item creations:  
        - For each product ID (`[].id`), creates an input object with:  
          - Associations: links to the new deal (`deal_id_created`) with association type ID 20 (HubSpot-defined).  
          - Properties: sets `hs_product_id` to product ID, `quantity` to `"1"`.  
      - Authentication: HubSpot App Token.  
    - Inputs: Product IDs from previous node.  
    - Outputs: Confirmation of created line items.  
    - Edge cases: API failures, association errors, invalid product IDs, or deal IDs.  
    - Version: 4.2  

#### 2.4 Notification

- **Overview:**  
Sends a Slack message to notify that the line item replication succeeded, including clickable links to the workflow and HubSpot deal records.

- **Nodes Involved:**  
  - Slack

- **Node Details:**

  - **Slack**  
    - Type: Slack node (message send)  
    - Role: Posts a confirmation message to a specific Slack channel.  
    - Configuration:  
      - Message text contains:  
        - White check mark emoji  
        - Link to the current n8n workflow with workflow ID and name  
        - Links to the HubSpot deals (won and created) using the extracted deal IDs.  
      - Channel selected by ID: `C051YHBJ1G8`.  
      - Does not include a link to the workflow automatically (option disabled).  
      - Credentials: Slack API with OAuth token.  
    - Inputs: Receives output from the line item creation node.  
    - Outputs: Slack message sent confirmation.  
    - Edge cases: Slack API rate limits, invalid channel ID, or token expired.  
    - Version: 2.2  

---

### 3. Summary Table

| Node Name                                   | Node Type           | Functional Role                             | Input Node(s)           | Output Node(s)                                | Sticky Note                                                                                                                                |
|---------------------------------------------|---------------------|---------------------------------------------|-------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                                     | Webhook Trigger     | Entry point, receives webhook with deal IDs | None                    | Retrieve deals Ids                            | **Step 1.** Triggered by HubSpot Workflow                                                                                                 |
| Retrieve deals Ids                          | Set                 | Parses and sets deal_id_won and deal_id_created | Webhook                 | Get deal won line items                       | **Step 2.** Set the Ids of the deal won and the deal created                                                                               |
| Get deal won line items                     | HTTP Request        | Retrieves line items associated with won deal | Retrieve deals Ids      | Get batch SKUs from line items                | **Step 3.** - Get line items IDs from the deal won - Retrieve the SKUs from those line items - Get product based on SKUs - Create new line items |
| Get batch SKUs from line items              | HTTP Request        | Fetches detailed line item info including SKUs | Get deal won line items | Get Batch Product IDs by SKUs                 | **Step 3.** - Get line items IDs from the deal won - Retrieve the SKUs from those line items - Get product based on SKUs - Create new line items |
| Get Batch Product IDs by SKUs               | HTTP Request        | Retrieves product details by SKU              | Get batch SKUs from line items | Create Batch line items based on productId and associate to deals | **Step 3.** - Get line items IDs from the deal won - Retrieve the SKUs from those line items - Get product based on SKUs - Create new line items |
| Create Batch line items based on productId and associate to deals | HTTP Request | Creates new line items linked to new deal using product IDs | Get Batch Product IDs by SKUs | Slack                                        | **Step 3.** - Get line items IDs from the deal won - Retrieve the SKUs from those line items - Get product based on SKUs - Create new line items |
| Slack                                       | Slack               | Sends Slack notification of success          | Create Batch line items based on productId and associate to deals | None                                         |                                                                                                                                             |
| Sticky Note                                 | Sticky Note         | Documentation and overview                    | None                    | None                                          | # Replicate Line Items on New Deal in HubSpot Workflow ... (full description as in node)                                                    |
| Sticky Note1                                | Sticky Note         | Step 1 comment                                | None                    | None                                          | **Step 1.** Triggered by HubSpot Workflow                                                                                                 |
| Sticky Note2                                | Sticky Note         | Step 2 comment                                | None                    | None                                          | **Step 2.** Set the Ids of the deal won and the deal created                                                                               |
| Sticky Note3                                | Sticky Note         | Step 3 comment                                | None                    | None                                          | **Step 3.** - Get line items IDs from the deal won - Retrieve the SKUs from those line items - Get product based on SKUs - Create new line items |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**
   - Type: Webhook  
   - Set the path to a unique identifier (e.g., `833df60e-a78f-4a59-8244-9694f27cf8ae`).  
   - Method: GET (default).  
   - No authentication required.  
   - This node will receive query parameters `deal_id_won` and `deal_id_created`.  

2. **Create a Set node named "Retrieve deals Ids":**
   - Connect it to the Webhook node.  
   - Add two assignments:  
     - `deal_id_won` (string): `={{ $json.query.deal_id_won }}`  
     - `deal_id_created` (string): `={{ $json.query.deal_id_created.match(/0-3-(\d+)$/)[1] }}`  
   - This extracts and cleans the deal IDs for later use.  

3. **Create an HTTP Request node named "Get deal won line items":**
   - Connect it to "Retrieve deals Ids".  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v4/associations/deals/line_items/batch/read`  
   - Body Content Type: JSON  
   - Body:  
     ```
     {
       "inputs": [
         {
           "id": "{{ $json.deal_id_won }}"
         }
       ]
     }
     ```  
   - Authentication: Use HubSpot App Token credentials.  
   - Credentials: Configure HubSpot App Token integration with your private app token.  

4. **Create an HTTP Request node named "Get batch SKUs from line items":**
   - Connect it to "Get deal won line items".  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v3/objects/line_items/batch/read`  
   - Query parameter: `archived=false`  
   - Body Content Type: JSON  
   - Body (expression):  
     ```
     {
       "idProperty": "hs_object_id",
       "inputs": $jmespath($json.results, `[].to[].{id: to_string(toObjectId)}`),
       "properties": [
         "hs_object_id",
         "name",
         "hs_sku"
       ]
     }
     ```  
   - Authentication: HubSpot App Token.  

5. **Create an HTTP Request node named "Get Batch Product IDs by SKUs":**
   - Connect it to "Get batch SKUs from line items".  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v3/objects/products/batch/read`  
   - Body Content Type: JSON  
   - Body (expression):  
     ```
     {
       "idProperty": "hs_sku",
       "inputs": $jmespath($json.results,"[].properties.{id:to_string(hs_sku)}"),
       "properties": [
         "idProperty",
         "name",
         "hs_object_id",
         "recurringbillingfrequency",
         "hs_price_eur"
       ]
     }
     ```  
   - Authentication: HubSpot App Token.  

6. **Create an HTTP Request node named "Create Batch line items based on productId and associate to deals":**
   - Connect it to "Get Batch Product IDs by SKUs".  
   - Method: POST  
   - URL: `https://api.hubapi.com/crm/v3/objects/line_items/batch/create`  
   - Body Content Type: JSON  
   - Body (expression):  
     ```
     {
       "inputs": $jmespath($json.results,"[].id")
         .map(id => ({
           "associations": [
             {
               "types": [
                 {
                   "associationCategory": "HUBSPOT_DEFINED",
                   "associationTypeId": 20
                 }
               ],
               "to": {
                 "id": $('Retrieve deals Ids').item.json["deal_id_created"]
               }
             }
           ],
           "properties": {
             "hs_product_id": id,
             "quantity": "1"
           }
         }))
     }
     ```  
   - Authentication: HubSpot App Token.  

7. **Create a Slack node named "Slack":**
   - Connect it to "Create Batch line items based on productId and associate to deals".  
   - Operation: Send Message  
   - Channel: Use channel ID `C051YHBJ1G8` (replace with your Slack channel ID if different).  
   - Message text (use expression):  
     ```
     =":white_check_mark: {{ `<https://arnaud-growth.app.n8n.cloud/workflow/${$workflow.id}|${$workflow.name}> successful on <https://app-eu1.hubspot.com/contacts/3418361/record/0-3/${$('Retrieve deals Ids').item.json["deal_id_won"]}|Deal won> and <https://app-eu1.hubspot.com/contacts/3418361/record/0-3/${$('Retrieve deals Ids').item.json["deal_id_created"]}|Deal created>` }}"
     ```  
   - Disable "Include Link To Workflow".  
   - Credentials: Configure Slack API OAuth2 credentials.  

8. **Configure Credentials:**
   - HubSpot App Token: Create a Private App in HubSpot, generate token, and add it in n8n under Credentials > HubSpot App Token.  
   - Slack API: Create Slack app, generate OAuth token with chat permissions, and add in n8n under Credentials > Slack API.  

9. **Optional: Add error workflow or error handling nodes to monitor and manage failures.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to be triggered from a HubSpot Deal Workflow that sends a webhook with `deal_id_won` and `deal_id_created`. | HubSpot Workflow Setup: Set trigger on deal stage = Won, create record (deal), then send webhook to n8n.            |
| HubSpot API endpoints used require appropriate scopes in HubSpot Private Apps for deals, line items, and products access.            | HubSpot Private Apps and Token Setup: https://developers.hubspot.com/docs/api/private-apps                          |
| Slack notifications help keep the team informed about automation success. Ensure Slack tokens have chat permissions.                 | Slack API Setup: https://api.slack.com/authentication/oauth-v2                                                     |
| The regex used to parse `deal_id_created` expects format `0-3-<number>`; adjust if your HubSpot record IDs differ.                   | Regular expression: `/0-3-(\d+)$/`                                                                                   |
| Consider adding error handling or retries to manage API rate limits and transient errors.                                             | n8n documentation on error workflows: https://docs.n8n.io/nodes/logic-nodes/error-trigger/                           |

---

This document provides a detailed and structured reference to fully understand, reproduce, and maintain the "Replicate Line Items on New Deal in HubSpot and notify with Slack" workflow.