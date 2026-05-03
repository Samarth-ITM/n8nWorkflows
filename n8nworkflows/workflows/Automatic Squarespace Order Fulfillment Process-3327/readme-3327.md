Automatic Squarespace Order Fulfillment Process

https://n8nworkflows.xyz/workflows/automatic-squarespace-order-fulfillment-process-3327


# Automatic Squarespace Order Fulfillment Process

### 1. Workflow Overview

This workflow automates the fulfillment process for Squarespace orders by marking pending orders as fulfilled without manual intervention. It is designed for Squarespace store owners who want to streamline order processing, especially those selling digital or personalized products requiring instant fulfillment.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initialization**: Starts the workflow manually or on a schedule and sets global parameters for API interaction.
- **1.2 Retrieve Pending Orders**: Uses an HTTP Request node to fetch all pending orders from Squarespace’s API, supporting pagination.
- **1.3 Order Splitting and Filtering**: Splits the fetched orders into individual items and filters them based on custom criteria (e.g., age of order).
- **1.4 Fulfillment Execution**: Loops over filtered orders and sends fulfillment requests to Squarespace to mark orders as fulfilled.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block initiates the workflow either manually or on a scheduled interval and sets global variables required for API calls, such as API version, date filters, pagination cursor, fulfillment status, and maximum pages to fetch.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger (Scheduled Trigger)  
  - Globals (Set node)  
  - Sticky Note3 (Instructional note)  
  - Sticky Note (Overview and setup instructions)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow.  
    - Configuration: No parameters; triggers workflow on user action.  
    - Inputs: None  
    - Outputs: Globals node  
    - Edge Cases: None specific; manual trigger depends on user action.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow on a recurring schedule (default is every minute).  
    - Configuration: Interval set to every minute (default empty interval object).  
    - Inputs: None  
    - Outputs: Globals node  
    - Edge Cases: None specific; ensure schedule frequency matches API rate limits.

  - **Globals**  
    - Type: Set  
    - Role: Defines global parameters for API requests and pagination control.  
    - Configuration:  
      - `api-version`: "1.0" (Squarespace API version)  
      - `modifiedAfter`: empty (ISO 8601 datetime filter)  
      - `modifiedBefore`: empty (ISO 8601 datetime filter)  
      - `cursor`: empty (pagination cursor)  
      - `fulfillmentStatus`: "PENDING" (filter orders by status)  
      - `maxPage`: -1 (enables infinite pagination)  
    - Inputs: Manual Trigger or Schedule Trigger  
    - Outputs: Query pending Orders node  
    - Edge Cases: If conflicting filters are set (e.g., cursor with date filters), API may reject requests.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Instruction to edit the Globals node parameters.  
    - Content: "## Edit this node 👇"  
    - Position: Near Globals node for user guidance.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides detailed setup instructions and overview of global parameters.  
    - Content: Explains API version, date filters, cursor usage, fulfillment status options, and maxPage usage.  
    - Position: Top-left corner for visibility.

---

#### 2.2 Retrieve Pending Orders

- **Overview:**  
  This block fetches all pending orders from Squarespace’s Commerce API using an HTTP Request node with support for pagination to retrieve multiple pages of orders.

- **Nodes Involved:**  
  - Query pending Orders (HTTP Request)  
  - Split Out Order (Split Out)  
  - Sticky Note1 (Filtering guidance)

- **Node Details:**

  - **Query pending Orders**  
    - Type: HTTP Request  
    - Role: Calls Squarespace Orders API to retrieve orders filtered by global parameters.  
    - Configuration:  
      - URL dynamically constructed using `api-version` from Globals node.  
      - Query parameters include `modifiedAfter`, `modifiedBefore`, `cursor`, and `fulfillmentStatus`.  
      - Pagination enabled using `nextPageCursor` from API response.  
      - Pagination settings:  
        - `maxRequests` set to infinite if `maxPage` = -1, else limited by `maxPage`.  
        - Pagination completes when no nextPageCursor is returned.  
      - Authentication: OAuth2 and HTTP Header API key (both configured).  
    - Inputs: Globals node  
    - Outputs: Split Out Order node  
    - Edge Cases:  
      - API rate limits or authentication failures.  
      - Invalid or conflicting query parameters.  
      - Pagination cursor misuse.  
      - Network timeouts.

  - **Split Out Order**  
    - Type: Split Out  
    - Role: Splits the array of orders in the `result` field of the API response into individual order items for processing.  
    - Configuration: Splits on the `result` field.  
    - Inputs: Query pending Orders node  
    - Outputs: Filter Orders node  
    - Edge Cases: Empty or malformed `result` field may cause no output.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides guidance on filtering orders for fulfillment.  
    - Content: Explains use cases for filtering (digital downloads, third-party fulfillment).  
    - Positioned near Filter Orders node for context.

---

#### 2.3 Order Splitting and Filtering

- **Overview:**  
  This block filters individual orders to ensure only valid orders are processed for fulfillment, based on custom criteria such as order age.

- **Nodes Involved:**  
  - Filter Orders (Filter)  
  - Sticky Note1 (Filtering guidance)

- **Node Details:**

  - **Filter Orders**  
    - Type: Filter  
    - Role: Filters orders that are older than 24 hours (to avoid immediate fulfillment of very recent orders).  
    - Configuration:  
      - Condition: `(Current time - order.createdOn) / (1000 * 60 * 60) > 24` hours  
      - Uses JavaScript expression to calculate order age in hours.  
    - Inputs: Split Out Order node  
    - Outputs: Loop Over Items node  
    - Edge Cases:  
      - Orders without `createdOn` field may fail expression evaluation.  
      - Timezone inconsistencies in `createdOn` may affect filtering.

---

#### 2.4 Fulfillment Execution

- **Overview:**  
  This block loops over filtered orders and sends a fulfillment request to Squarespace API to mark each order as fulfilled, optionally sending notifications to customers.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches)  
  - Fulfill Order (HTTP Request)  
  - Sticky Note4 (Fulfillment instructions)

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes orders one by one or in batches to control API request flow.  
    - Configuration: Default batch size (not explicitly set, defaults to 1).  
    - Inputs: Filter Orders node  
    - Outputs: Fulfill Order node (main output), and an empty output (unused).  
    - Edge Cases: Large order volumes may require batch size tuning to avoid rate limits.

  - **Fulfill Order**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Squarespace API to fulfill the order identified by its `id`.  
    - Configuration:  
      - URL dynamically constructed using `api-version` from Globals and order `id` from Filter Orders node.  
      - HTTP Method: POST  
      - Request Body: JSON `{ "shouldSendNotification": true }` to notify customers upon fulfillment.  
      - Authentication: OAuth2 and HTTP Header API key (both configured).  
    - Inputs: Loop Over Items node  
    - Outputs: Loop Over Items node (for next batch)  
    - Edge Cases:  
      - API errors such as invalid order ID, authentication failure, or network issues.  
      - Rate limiting if too many fulfillments processed rapidly.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Provides a link to Squarespace API documentation for fulfilling orders and notes about sending notifications.  
    - Content: "[Fulfill an order](https://developers.squarespace.com/commerce-apis/fulfill-order) - `shouldSendNotification` to send notifications to customer"  
    - Positioned near Fulfill Order node.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                         | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                          |
|---------------------|--------------------|---------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger     | Manual start of workflow               | None                     | Globals                  |                                                                                                    |
| Schedule Trigger     | Schedule Trigger   | Scheduled start of workflow            | None                     | Globals                  |                                                                                                    |
| Globals             | Set                | Defines global API and pagination params| On clicking 'execute', Schedule Trigger | Query pending Orders     | Edit this node 👇                                                                                   |
| Sticky Note3         | Sticky Note        | Instruction to edit Globals node       | None                     | None                     | Edit this node 👇                                                                                   |
| Sticky Note          | Sticky Note        | Setup overview and instructions        | None                     | None                     | Explains API version, filters, cursor, fulfillmentStatus, maxPage                                  |
| Query pending Orders | HTTP Request       | Fetches pending orders from Squarespace| Globals                   | Split Out Order          |                                                                                                    |
| Split Out Order      | Split Out          | Splits orders array into individual orders| Query pending Orders      | Filter Orders            |                                                                                                    |
| Filter Orders        | Filter             | Filters orders older than 24 hours     | Split Out Order           | Loop Over Items          | Filtering orders for fulfillment 👇 You exclusively sell digital downloads or gift cards, etc.     |
| Loop Over Items      | Split In Batches   | Processes orders in batches             | Filter Orders             | Fulfill Order            |                                                                                                    |
| Fulfill Order        | HTTP Request       | Sends fulfillment request to Squarespace| Loop Over Items           | Loop Over Items          | Create fulfillment 👇 [Fulfill an order](https://developers.squarespace.com/commerce-apis/fulfill-order) |
| Sticky Note1         | Sticky Note        | Guidance on filtering orders            | None                     | None                     | Filtering orders for fulfillment 👇 You exclusively sell digital downloads or gift cards, etc.     |
| Sticky Note4         | Sticky Note        | Fulfillment API instructions            | None                     | None                     | Create fulfillment 👇 [Fulfill an order](https://developers.squarespace.com/commerce-apis/fulfill-order) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "On clicking 'execute'"  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Schedule Trigger Node**  
   - Name: "Schedule Trigger"  
   - Type: Schedule Trigger  
   - Set interval to desired frequency (default every minute).

3. **Create Set Node for Globals**  
   - Name: "Globals"  
   - Type: Set  
   - Add fields:  
     - `api-version` (string): "1.0"  
     - `modifiedAfter` (string): "" (empty)  
     - `modifiedBefore` (string): "" (empty)  
     - `cursor` (string): "" (empty)  
     - `fulfillmentStatus` (string): "PENDING"  
     - `maxPage` (number): -1 (for infinite pagination)  
   - Connect outputs of both triggers to this node.

4. **Create HTTP Request Node to Query Orders**  
   - Name: "Query pending Orders"  
   - Type: HTTP Request  
   - URL: `https://api.squarespace.com/{{ $json["api-version"] }}/commerce/orders`  
   - Method: GET  
   - Authentication:  
     - OAuth2 (Squarespace OAuth 2.0 credential)  
     - HTTP Header Auth (Squarespace API key credential)  
   - Query Parameters:  
     - `modifiedAfter` = `={{ $json.modifiedAfter }}`  
     - `modifiedBefore` = `={{ $json.modifiedBefore }}`  
     - `cursor` = `={{ $json.cursor }}`  
     - `fulfillmentStatus` = `={{ $json.fulfillmentStatus }}`  
   - Pagination:  
     - Enable pagination with parameters:  
       - Cursor parameter name: `cursor`  
       - Cursor value: `={{ $response.body.pagination.nextPageCursor }}`  
       - Max requests: `={{ $json.maxPage === -1 ? Infinity : $json.maxPage }}`  
       - Complete expression: `={{ !$response.body.pagination.nextPageCursor }}`  
   - Connect Globals node output to this node.

5. **Create Split Out Node**  
   - Name: "Split Out Order"  
   - Type: Split Out  
   - Field to split out: `result` (array of orders)  
   - Connect Query pending Orders node output to this node.

6. **Create Filter Node**  
   - Name: "Filter Orders"  
   - Type: Filter  
   - Condition:  
     - Expression: `(new Date().getTime() - new Date($json.createdOn).getTime()) / (1000 * 60 * 60) > 24`  
     - This filters orders older than 24 hours.  
   - Connect Split Out Order node output to this node.

7. **Create Split In Batches Node**  
   - Name: "Loop Over Items"  
   - Type: Split In Batches  
   - Default batch size (1) or configure as needed.  
   - Connect Filter Orders node output to this node.

8. **Create HTTP Request Node to Fulfill Order**  
   - Name: "Fulfill Order"  
   - Type: HTTP Request  
   - URL: `https://api.squarespace.com/{{ $('Globals').item.json["api-version"] }}/commerce/orders/{{ $('Filter Orders').item.json.id }}/fulfillments`  
   - Method: POST  
   - Authentication:  
     - OAuth2 (Squarespace OAuth 2.0 credential)  
     - HTTP Header Auth (Squarespace API key credential)  
   - Body Content-Type: JSON  
   - Body: `{ "shouldSendNotification": true }`  
   - Connect Loop Over Items node main output to this node.  
   - Connect Fulfill Order node output back to Loop Over Items node to continue batch processing.

9. **Add Sticky Notes for Guidance (Optional)**  
   - Add notes near Globals node explaining parameter setup.  
   - Add notes near Filter Orders node explaining filtering rationale.  
   - Add notes near Fulfill Order node with link to Squarespace API documentation.

10. **Configure Credentials**  
    - Create and configure Squarespace OAuth2 credentials with client ID, secret, and token URL.  
    - Create HTTP Header Auth credentials with Squarespace API key.  
    - Assign these credentials to HTTP Request nodes accordingly.

11. **Test Workflow**  
    - Run manually via "On clicking 'execute'" or wait for scheduled trigger.  
    - Monitor logs for errors such as authentication failures, API errors, or empty results.  
    - Adjust Globals parameters as needed for date filters or pagination.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Squarespace API Key is required, retrieve it from Squarespace settings: https://developers.squarespace.com/commerce-apis/authentication-and-permissions | Credentials requirement                                                                                |
| Fulfill an order API documentation: https://developers.squarespace.com/commerce-apis/fulfill-order             | Linked in Sticky Note4 near Fulfill Order node                                                      |
| Explore more Squarespace automation templates on n8n: https://n8n.io/creators/bangank36/                       | Additional workflow templates for Squarespace                                                       |
| Workflow supports infinite pagination by setting maxPage to -1 in Globals node                                 | Important for fetching all orders without manual pagination control                                  |
| Filtering orders older than 24 hours helps avoid premature fulfillment of very recent orders                   | Filtering rationale                                                                                  |
| The workflow supports both manual and scheduled execution                                                      | Flexibility in triggering the workflow                                                              |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and modifying the "Automatic Squarespace Order Fulfillment Process" workflow, enabling efficient automation of Squarespace order fulfillment.