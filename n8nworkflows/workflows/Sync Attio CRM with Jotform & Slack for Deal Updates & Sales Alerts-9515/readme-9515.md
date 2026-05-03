Sync Attio CRM with Jotform & Slack for Deal Updates & Sales Alerts

https://n8nworkflows.xyz/workflows/sync-attio-crm-with-jotform---slack-for-deal-updates---sales-alerts-9515


# Sync Attio CRM with Jotform & Slack for Deal Updates & Sales Alerts

### 1. Workflow Overview

This workflow automates synchronization between Jotform form submissions, Attio CRM, and Slack notifications to streamline deal management and sales alerts. It is designed for sales teams or CRM administrators who want to capture leads from Jotform, ensure CRM data integrity, manage deal stages automatically, and alert their team via Slack.

Logical blocks:

- **1.1 Input Reception:** Captures form submission data from Jotform via webhook.
- **1.2 Data Preparation:** Formats and structures the input data and CRM reference values such as statuses and columns.
- **1.3 CRM Schema Verification:** Checks and ensures that required Deal stages ("Pending", "Urgent") and People attribute columns ("Message") exist in Attio CRM; creates them if missing.
- **1.4 Customer Lookup and Deal Logic:** Searches for the customer in Attio CRM by email, analyzes if a deal exists, and branches logic accordingly:
  - If deal exists → update deal stage to "Urgent".
  - If customer exists but no deal → create a new deal in "Pending".
  - If customer does not exist → create customer and a new deal in "Pending".
- **1.5 Slack Notification:** Sends a Slack message notifying the sales team about the newly created or updated deal.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming form submissions from Jotform via a webhook.
- **Nodes Involved:** Receive form submissions
- **Node Details:**

  - **Receive form submissions**
    - Type: Webhook
    - Role: Entry point capturing HTTP POST requests at path `/events`.
    - Config: Uses POST method, webhook ID assigned.
    - Inputs: External Jotform submissions.
    - Outputs: JSON data containing form fields (name, email, message).
    - Edge cases: Missing or malformed webhook data, webhook not triggered.
    - Version: 2.1

#### 2.2 Data Preparation

- **Overview:** Sets structured fields and prepares static data for CRM schema references and customer data extraction.
- **Nodes Involved:** Edit Fields
- **Node Details:**

  - **Edit Fields**
    - Type: Set
    - Role: Defines static keys and values for statuses (Pending, Urgent), column definitions (Message), and extracts customer info from incoming submission.
    - Config: Uses JSON mode with expressions to map input JSON fields (`body.name`, `body.email`, `body.message`) into structured customer object.
    - Inputs: Output from webhook node.
    - Outputs: JSON with structured `statuses`, `columns`, `customer`, and empty `data_tables_id` and `owner_email`.
    - Edge cases: Missing input fields, expression evaluation failures.
    - Version: 3.4

#### 2.3 CRM Schema Verification

- **Overview:** Ensures the CRM data tables have required deal stages and attribute columns. If missing, creates them in Attio CRM and records them in the local data table for persistence.
- **Nodes Involved:**  
  - If pending status does not exist  
  - If urgent status does not exist  
  - If message column does not exist  
  - If deals id does not exist  
  - If people id does not exist  
  - Get the deals id  
  - Get the deals id1  
  - Get the people id  
  - Get the deals id (CRM)  
  - Get the people id (CRM)  
  - Add deals id to DB  
  - Add pending status to CRM  
  - Add pending status to DB  
  - Add urgent status to CRM  
  - Add urgent status to DB  
  - Add message column to CRM  
  - Add message column to DB  

- **Node Details (selected examples):**

  - **If pending status does not exist**
    - Type: DataTable (check row existence)
    - Role: Checks if "Pending" status key exists in the data table.
    - Input: From Edit Fields node.
    - Output: Routes flow if status missing.
    - Edge cases: Data table ID undefined.
    - Version: 1

  - **Get the deals id (CRM)**
    - Type: HTTP Request
    - Role: Fetches deal object ID from Attio CRM API.
    - Config: GET to `https://api.attio.com/v2/objects/deals` with Bearer Auth.
    - Inputs: From If deals id does not exist.
    - Outputs: JSON with deals object ID.
    - Edge cases: Auth failure, API downtime.
    - Version: 4.2

  - **Add pending status to CRM**
    - Type: HTTP Request
    - Role: Creates "Pending" status in Attio CRM deal stages.
    - Config: POST to `https://api.attio.com/v2/objects/{{deals_id}}/attributes/stage/statuses` with JSON body specifying title.
    - Inputs: From Get the deals id.
    - Outputs: Confirmation JSON.
    - Edge cases: API rate limits, invalid data.
    - Version: 4.2

  - **Add pending status to DB**
    - Type: DataTable
    - Role: Records the newly added "Pending" status in local data table.
    - Inputs: From Add pending status to CRM.
    - Outputs: Confirmation.
    - Edge cases: Data table write error.
    - Version: 1

  (Similar logic applies for `urgent` status and `message` column, as well as storing IDs for deals and people.)

#### 2.4 Customer Lookup and Deal Logic

- **Overview:** Searches for the customer in Attio CRM by email, then processes deal creation or update accordingly.
- **Nodes Involved:**  
  - Get customer by email  
  - Prepare customer data  
  - Check if the customer is in the CRM (If node)  
  - Switch  
  - Update the deal  
  - Create a new deal  
  - Add the customer to the CRM  

- **Node Details:**

  - **Get customer by email**
    - Type: HTTP Request
    - Role: Queries Attio CRM for people records matching the submitted email.
    - Config: POST to `https://api.attio.com/v2/objects/people/records/query` with filter JSON by email.
    - Input: Output of Edit Fields.
    - Output: Customer data if found.
    - Edge cases: No results, auth failure.
    - Version: 4.2

  - **Prepare customer data**
    - Type: Code
    - Role: Parses API response to check if customer exists, extracts name, email, and deal details if any.
    - Config: JavaScript code processing input JSON.
    - Input: From Get customer by email.
    - Output: Structured customer object with flags `exists` and `deal.exists`.
    - Edge cases: Unexpected API response structure.
    - Version: 2

  - **Check if the customer is in the CRM**
    - Type: If
    - Role: Branches workflow based on whether customer exists.
    - Config: Checks boolean flag from previous node.
    - Edges: True branch → Switch; False branch → Add the customer to the CRM.
    - Version: 2.2

  - **Switch**
    - Type: Switch
    - Role: Further branches based on if the customer has an associated deal.
    - Config: Checks `deal.exists` boolean.
    - Outputs:
      - "Deal exists" → Update the deal node.
      - "Deal doesn't exist" → Create a new deal node.
    - Version: 3.3

  - **Update the deal**
    - Type: HTTP Request
    - Role: Updates existing deal stage to "Urgent".
    - Config: PATCH to `https://api.attio.com/v2/objects/deals/records/{{deal_id}}` with JSON body setting stage.
    - Input: Switch node's "Deal exists" output.
    - Output: Proceeds to Slack notification.
    - Edge cases: Deal ID invalid, API fail.
    - Version: 4.2

  - **Create a new deal**
    - Type: HTTP Request
    - Role: Creates a new deal in "Pending" stage, associated with customer.
    - Config: POST to Attio deals records endpoint with JSON body including customer name, stage, owner email, and associated people by email.
    - Input: Switch node's "Deal doesn't exist" output or Add the customer to the CRM.
    - Output: Proceeds to Slack notification.
    - Edge cases: API validation failure, missing owner email.
    - Version: 4.2

  - **Add the customer to the CRM**
    - Type: HTTP Request
    - Role: Creates a new person record in Attio CRM with name, email, and message attribute.
    - Config: POST to Attio people records endpoint with JSON body.
    - Input: From If node's False branch (customer not found).
    - Output: Leads to Create a new deal.
    - Edge cases: Duplicate records, API failure.
    - Version: 4.2

#### 2.5 Slack Notification

- **Overview:** Sends a Slack message to notify the team about deal creation or update.
- **Nodes Involved:** Send slack message
- **Node Details:**

  - **Send slack message**
    - Type: HTTP Request
    - Role: Sends a POST request to Slack webhook or API with message text.
    - Config: POST with JSON body containing message text. Uses conditional expression to set message depending on deal creation or update.
    - Input: From Update the deal or Create a new deal nodes.
    - Output: None (end of workflow).
    - Edge cases: Slack API failure, invalid webhook URL.
    - Version: 4.2

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                                   | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                      |
|-------------------------------|----------------------|-------------------------------------------------|------------------------------|-------------------------------------|-----------------------------------------------------------------|
| Receive form submissions       | Webhook              | Entry point to capture Jotform submissions       | External                      | Edit Fields                         |                                                                 |
| Edit Fields                   | Set                  | Prepare static CRM keys and extract customer data| Receive form submissions      | If pending status does not exist, If urgent status does not exist, If message column does not exist, If deals id does not exist, If people id does not exist, Get customer by email |                                                                 |
| If pending status does not exist | DataTable            | Check if "Pending" status exists in data table   | Edit Fields                  | Get the deals id                    | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| If urgent status does not exist | DataTable            | Check if "Urgent" status exists                   | Edit Fields                  | Get the deals id1                   | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| If message column does not exist | DataTable            | Check if "Message" column exists                   | Edit Fields                  | Get the people id                   | Step 2: Make sure that Message attribute column is added to People |
| If deals id does not exist      | DataTable            | Check if deals object ID exists                    | Edit Fields                  | Get the deals id (CRM)              | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| If people id does not exist     | DataTable            | Check if people object ID exists                   | Edit Fields                  | Get the people id (CRM)             | Step 2: Make sure that Message attribute column is added to People |
| Get the deals id               | DataTable            | Retrieve deals object ID                           | If deals id does not exist    | Add pending status to CRM           | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Get the deals id1              | DataTable            | Retrieve deals object ID                           | If urgent status does not exist | Add urgent status to CRM           | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Get the people id              | DataTable            | Retrieve people object ID                          | If message column does not exist | Add message column to CRM          | Step 2: Make sure that Message attribute column is added to People |
| Get the deals id (CRM)         | HTTP Request         | Fetch deals object ID from Attio API              | If deals id does not exist    | Add deals id to DB                  | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Get the people id (CRM)        | HTTP Request         | Fetch people object ID from Attio API              | If people id does not exist   | Add people id to DB                 | Step 2: Make sure that Message attribute column is added to People |
| Add deals id to DB             | DataTable            | Store deals object ID locally                      | Get the deals id (CRM)        | Get the deals id                   | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Add pending status to CRM      | HTTP Request         | Create "Pending" stage in Attio CRM                | Get the deals id              | Add pending status to DB            | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Add pending status to DB       | DataTable            | Store "Pending" status locally                      | Add pending status to CRM     | None                             | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Add urgent status to CRM       | HTTP Request         | Create "Urgent" stage in Attio CRM                  | Get the deals id1             | Add urgent status to DB             | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Add urgent status to DB        | DataTable            | Store "Urgent" status locally                        | Add urgent status to CRM      | None                             | Step 1: Make sure that Pending and Urgent stages are added to Deals |
| Add message column to CRM      | HTTP Request         | Create "Message" column attribute in Attio CRM     | Get the people id             | Add message column to DB            | Step 2: Make sure that Message attribute column is added to People |
| Add message column to DB       | DataTable            | Store "Message" column locally                       | Add message column to CRM     | None                             | Step 2: Make sure that Message attribute column is added to People |
| Get customer by email          | HTTP Request         | Query Attio people by email                          | Edit Fields                  | Prepare customer data               |                                                                 |
| Prepare customer data          | Code                 | Parse response, check existence, extract details   | Get customer by email         | Check if the customer is in the CRM|                                                                 |
| Check if the customer is in the CRM | If                   | Branch based on customer existence                   | Prepare customer data         | Switch (if exists), Add the customer to the CRM (if not) |                                                                 |
| Switch                       | Switch                | Branch based on deal existence                        | Check if the customer is in the CRM | Update the deal (deal exists), Create a new deal (deal does not exist) |                                                                 |
| Update the deal               | HTTP Request          | Update deal stage to "Urgent"                         | Switch (deal exists)          | Send slack message                 |                                                                 |
| Create a new deal             | HTTP Request          | Create new deal in "Pending" stage                     | Switch (deal does not exist), Add the customer to the CRM | Send slack message                 |                                                                 |
| Add the customer to the CRM    | HTTP Request          | Create a new person record in Attio CRM               | Check if the customer is in the CRM (False branch) | Create a new deal                  |                                                                 |
| Send slack message            | HTTP Request          | Notify team about deal creation or update              | Update the deal, Create a new deal | None                             |                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node:**  
   - Name: `Receive form submissions`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `events`  
   - Purpose: Receive Jotform submissions with fields name, email, message.

2. **Add Set node:**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Mode: Raw JSON  
   - Define JSON with static keys for statuses and columns, and map customer data using expressions from webhook input:  
     ```json
     {
       "data_tables_id": "",
       "statuses": {
         "pending": {"key": "PENDING_STATUS", "value": "Pending"},
         "urgent": {"key": "URGENT_STATUS", "value": "Urgent"}
       },
       "columns": {
         "message": {
           "key": "MESSAGE_COLUMN",
           "value": "Message",
           "slug": "message",
           "description": "the customer message",
           "type": "text",
           "is_required": false,
           "is_unique": false,
           "is_multiselect": false
         }
       },
       "customer": {
         "name": "={{ $json.body.name }}",
         "email": "={{ $json.body.email }}",
         "message": "={{ $json.body.message }}"
       },
       "owner_email": ""
     }
     ```
   - Connect `Receive form submissions` → `Edit Fields`.

3. **Add DataTable nodes to verify schema existence:**  
   - Nodes:  
     - `If pending status does not exist` (check rowNotExists for key PENDING_STATUS)  
     - `If urgent status does not exist` (check rowNotExists for key URGENT_STATUS)  
     - `If message column does not exist` (check rowNotExists for key MESSAGE_COLUMN)  
     - `If deals id does not exist` (check rowNotExists for key DEALS_ID)  
     - `If people id does not exist` (check rowNotExists for key PEOPLE_ID)  
   - All receive input from `Edit Fields`.

4. **Add HTTP Request nodes to fetch CRM object IDs:**  
   - `Get the deals id (CRM)` and `Get the people id (CRM)`  
   - Use Attio API endpoints `/v2/objects/deals` and `/v2/objects/people` respectively.  
   - Use Bearer authentication with Attio Access Token credentials.

5. **Add DataTable nodes to store fetched IDs:**  
   - `Add deals id to DB` and `Add people id to DB`  
   - Connected from the respective HTTP request nodes.

6. **Add HTTP Request nodes to create statuses and columns if missing:**  
   - `Add pending status to CRM` and `Add urgent status to CRM` (POST to `/attributes/stage/statuses` endpoint with JSON body for title)  
   - `Add message column to CRM` (POST to `/attributes` endpoint with JSON describing the Message column)  
   - Connected from the respective DataTable existence checks and ID retrieval nodes.

7. **Add DataTable nodes to record created statuses and columns:**  
   - `Add pending status to DB`, `Add urgent status to DB`, `Add message column to DB`  
   - Connected from respective CRM creation HTTP requests.

8. **Add HTTP Request node to query customer by email:**  
   - `Get customer by email`  
   - POST to `/v2/objects/people/records/query` with filter JSON by email.  
   - Input: output of `Edit Fields`.

9. **Add Code node to parse customer data:**  
   - `Prepare customer data`  
   - JavaScript code to check if customer exists, extract name, email, and deal details.

10. **Add If node to check if customer exists:**  
    - `Check if the customer is in the CRM`  
    - Condition: `$json.exists == true`.

11. **Add Switch node to branch on deal existence:**  
    - `Switch` node  
    - Condition on `$json.deal.exists` boolean.

12. **Add HTTP Request nodes for deal operations:**  
    - `Update the deal` (PATCH to update stage to Urgent)  
    - `Create a new deal` (POST to create deal with Pending stage and associate with customer)  
    - `Add the customer to the CRM` (POST to create new person record with message attribute)  

13. **Connect nodes according to logic:**  
    - `Check if the customer is in the CRM` false branch → `Add the customer to the CRM` → `Create a new deal`  
    - `Check` true branch → `Switch`  
    - `Switch` "Deal exists" → `Update the deal`  
    - `Switch` "Deal doesn't exist" → `Create a new deal`

14. **Add HTTP Request node to send Slack notifications:**  
    - `Send slack message`  
    - POST to Slack webhook or API URL  
    - JSON body with text message indicating if deal was created or updated.  
    - Connect from both `Update the deal` and `Create a new deal`.

15. **Configure credentials:**  
    - Attio Access Token credential for all HTTP requests to Attio API.  
    - Slack webhook or API credentials if required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow streamlines lead management by integrating Jotform, Attio CRM, and Slack notifications to automate deal updates and alerts, reducing manual follow-ups and improving sales responsiveness.                        | Workflow purpose                                                    |
| Step 1: Ensure Pending and Urgent deal stages exist in Attio CRM before running workflow.                                                                                                                                       | Sticky Note at positions near schema checks                        |
| Step 2: Ensure Message attribute column is added to People object in Attio CRM.                                                                                                                                                | Sticky Note near attribute column checks                          |
| Step 3: Workflow logic overview for customer and deal creation/update, with Slack notifications.                                                                                                                               | Sticky Note near main processing branch                           |
| Demo video showing the workflow in action: https://www.youtube.com/watch?v=uccGQWvLhkA                                                                                                                                          | Video link in sticky note                                          |
| Template usage video: https://www.youtube.com/watch?v=FCGmVZsGYWk                                                                                                                                                               | Video link in sticky note                                          |

---

**Disclaimer:** The text above is based solely on an automated n8n workflow integration. It complies with current content policies and does not contain illegal or offensive content. All data handled is legal and public.