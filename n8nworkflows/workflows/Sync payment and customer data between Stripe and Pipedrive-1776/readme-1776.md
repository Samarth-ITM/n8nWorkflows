Sync payment and customer data between Stripe and Pipedrive

https://n8nworkflows.xyz/workflows/sync-payment-and-customer-data-between-stripe-and-pipedrive-1776


# Sync payment and customer data between Stripe and Pipedrive

### 1. Workflow Overview

This workflow is designed to synchronize payment and customer data from Stripe with Pipedrive organizations daily. It extracts successful payment charges from Stripe, fetches corresponding customer details, merges this information, and then logs payment data as notes attached to the relevant organizations in Pipedrive. The workflow is suitable for organizations wanting to keep their CRM updated with financial transaction details automatically.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Execution Time Management:** Schedule the workflow to run daily and track time of last execution to query incremental data.
- **1.2 Stripe Payment Data Retrieval:** Search for successful payment charges created since the last run in Stripe.
- **1.3 Payment Data Preparation:** Split payment data array into individual items for processing.
- **1.4 Stripe Customer Data Retrieval:** Retrieve all customers from Stripe.
- **1.5 Customer Data Refinement:** Rename and filter customer fields to essentials.
- **1.6 Data Merging:** Combine payment charges with corresponding customer details.
- **1.7 Pipedrive Integration:** Search for corresponding organizations in Pipedrive and append payment notes.
- **1.8 Execution Timestamp Update:** Update stored timestamp for next incremental run.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Execution Time Management

**Overview:**  
This block initiates the workflow once daily at 8 a.m. It also manages the retrieval and update of the last execution timestamp to enable incremental data fetching from Stripe.

**Nodes Involved:**  
- Every day at 8 am (Cron)  
- Get last execution timestamp (FunctionItem)  
- Set new last execution timestamp (FunctionItem)

**Node Details:**

- **Every day at 8 am**  
  - Type: Cron Trigger  
  - Configuration: Triggers once per day at 8:00 a.m. (local time)  
  - Inputs: None (start node)  
  - Outputs: Passes control to "Get last execution timestamp"  
  - Edge Cases: Workflow does not run if n8n instance is down at trigger time.

- **Get last execution timestamp**  
  - Type: FunctionItem  
  - Role: Reads a globally stored static data value `lastExecution` (in Unix timestamp seconds). If none exists, initializes it to current timestamp. Adds two fields to each item: `executionTimeStamp` (current time) and `lastExecution` (stored last run time).  
  - Inputs: From Cron node  
  - Outputs: Passes enhanced item to Stripe charge search  
  - Edge Cases: Static data persistence failure could cause re-processing of older data or missing data.

- **Set new last execution timestamp**  
  - Type: FunctionItem  
  - Role: Updates the global static data `lastExecution` to the current run’s timestamp obtained from "Get last execution timestamp" node. Executes only once per workflow run (`executeOnce` enabled).  
  - Inputs: From "Create note with charge information" node  
  - Outputs: None further; end of workflow  
  - Edge Cases: Failure to update static data may cause duplicate or missed payments in future runs.

---

#### 1.2 Stripe Payment Data Retrieval

**Overview:**  
Queries Stripe for all successful payment charges created after the last workflow execution timestamp.

**Nodes Involved:**  
- Search for charges in Stripe (HTTP Request)  
- Split array of charges to items (Item Lists)

**Node Details:**

- **Search for charges in Stripe**  
  - Type: HTTP Request (Stripe API)  
  - Role: Executes a search query on Stripe’s charges endpoint using the last execution time filter with query: `created>{lastExecution} AND status:"succeeded"`.  
  - Authentication: Uses predefined Stripe credentials  
  - Inputs: From "Get last execution timestamp" node (provides lastExecution)  
  - Outputs: JSON array of charge objects under `data` property  
  - Edge Cases: API rate limits, authentication failures, network errors, or no new charges found (empty data array).

- **Split array of charges to items**  
  - Type: Item Lists  
  - Role: Splits the `data` array from the HTTP response into separate workflow items, each representing one charge.  
  - Inputs: From "Search for charges in Stripe"  
  - Outputs: Individual charge items for downstream processing  
  - Edge Cases: Empty input array results in no output items.

---

#### 1.3 Stripe Customer Data Retrieval

**Overview:**  
Retrieves all customers from Stripe to later match with payment charges.

**Nodes Involved:**  
- Get customers (Stripe node)  
- Rename fields and keep only needed fields (Set node)

**Node Details:**

- **Get customers**  
  - Type: Stripe node (native in n8n)  
  - Role: Retrieves all customers from Stripe with no filters, returning full customer data.  
  - Authentication: Uses Stripe credentials  
  - Inputs: None (triggered after splitting charges)  
  - Outputs: Array of customer objects  
  - Edge Cases: Large number of customers may affect performance; API rate limits; auth errors.

- **Rename fields and keep only needed fields**  
  - Type: Set node  
  - Role: Transforms customer data: renames `name` to `customerName`, `id` to `customerId` and discards all other fields to optimize merging.  
  - Inputs: From "Get customers"  
  - Outputs: Streamlined customer data items  
  - Edge Cases: Missing `name` or `id` fields in customer data may cause merge issues downstream.

---

#### 1.4 Data Merging

**Overview:**  
Merges payment charges with their corresponding customer details by matching customer IDs.

**Nodes Involved:**  
- Add customer name to charge data (Merge node)  
- Search organisation (Pipedrive node)  
- Add organisation Information to charge data (Merge node)  
- Create note with charge information (Pipedrive node)

**Node Details:**

- **Add customer name to charge data**  
  - Type: Merge node  
  - Role: Performs a key-based merge of charge items (input 1) with customer items (input 2) using keys `"customer"` (in charge data) and `"customerId"` (in customer data) to enrich charges with customer names.  
  - Inputs:  
    - Input 1: Charges (with `customer` field)  
    - Input 2: Customers (with `customerId`)  
  - Outputs: Charge objects augmented with customer names  
  - Edge Cases: Charges without matching customer data won't be merged; missing keys cause failed merges.

- **Search organisation**  
  - Type: Pipedrive node (Search operation on organization resource)  
  - Role: Searches Pipedrive for organizations matching the customer name extracted from merged charge data.  
  - Inputs: From "Add customer name to charge data"  
  - Outputs: Organization search results  
  - Edge Cases: No matching organizations found; API errors; rate limits.

- **Add organisation Information to charge data**  
  - Type: Merge node  
  - Role: Inner join merge by index between merged charge/customer data and Pipedrive organization search results, effectively attaching organization details to each charge item.  
  - Inputs:  
    - Input 1: Merged charge/customer data  
    - Input 2: Organization search results  
  - Outputs: Charge data enriched with organization information  
  - Edge Cases: Non-matching indices cause data loss due to inner join; inconsistencies in ordering.

- **Create note with charge information**  
  - Type: Pipedrive node (Create operation on note resource)  
  - Role: Creates a note attached to the organization in Pipedrive containing the charge description and formatted amount with currency.  
  - Inputs: From "Add organisation Information to charge data"  
  - Outputs: Confirmation of note creation  
  - Edge Cases: Missing organization IDs; API errors; rate limits; data formatting errors.

---

### 3. Summary Table

| Node Name                         | Node Type          | Functional Role                           | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                   |
|----------------------------------|--------------------|-----------------------------------------|----------------------------------|---------------------------------------------|---------------------------------------------------------------|
| Every day at 8 am                | Cron               | Triggers workflow daily at 8 a.m.       | None                             | Get last execution timestamp                 |                                                               |
| Get last execution timestamp     | FunctionItem       | Retrieves last run timestamp or sets it | Every day at 8 am                | Search for charges in Stripe                  |                                                               |
| Search for charges in Stripe     | HTTP Request       | Searches Stripe charges after last run  | Get last execution timestamp     | Split array of charges to items               |                                                               |
| Split array of charges to items  | Item Lists         | Splits Stripe charges array to items    | Search for charges in Stripe     | Add customer name to charge data              |                                                               |
| Get customers                   | Stripe             | Retrieves all Stripe customers           | None (runs after split charges)  | Rename fields and keep only needed fields     |                                                               |
| Rename fields and keep only needed fields | Set        | Keeps only customerName and customerId  | Get customers                   | Add customer name to charge data              |                                                               |
| Add customer name to charge data | Merge              | Merges charges with customer data       | Split array of charges, Rename fields | Search organisation, Add organisation Information to charge data |                                                               |
| Search organisation             | Pipedrive          | Searches for organization by customerName | Add customer name to charge data | Add organisation Information to charge data  |                                                               |
| Add organisation Information to charge data | Merge        | Inner join charges with organization data | Add customer name to charge data, Search organisation | Create note with charge information          |                                                               |
| Create note with charge information | Pipedrive       | Creates note on organization with charge info | Add organisation Information to charge data | Set new last execution timestamp              |                                                               |
| Set new last execution timestamp | FunctionItem       | Updates static lastExecution timestamp  | Create note with charge information | None                                       |                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron node "Every day at 8 am"**  
   - Type: Cron  
   - Set trigger to daily at 08:00 hours.

2. **Create FunctionItem node "Get last execution timestamp"**  
   - Connect output of Cron node to this node.  
   - Add code to read `lastExecution` from global static data or initialize to current timestamp.  
   - Add fields: `executionTimeStamp` (current time), `lastExecution` (saved time).  
   - Return the item.

3. **Create HTTP Request node "Search for charges in Stripe"**  
   - Connect output of "Get last execution timestamp" to this node.  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/charges/search`  
   - Authentication: Use Stripe API credentials (OAuth or API Key)  
   - Query Parameters:  
     - `query`: `created>{{$json["lastExecution"]}} AND status:"succeeded"`  
   - Ensure the response contains a `data` array with charges.

4. **Create Item Lists node "Split array of charges to items"**  
   - Connect output of HTTP Request node to this node.  
   - Set `fieldToSplitOut` to `data` to split charges array to individual items.

5. **Create Stripe node "Get customers"**  
   - Connect output of "Split array of charges to items" to this node (to run after charges are split).  
   - Resource: Customer  
   - Operation: Get All  
   - Return All: True  
   - Use Stripe credentials.

6. **Create Set node "Rename fields and keep only needed fields"**  
   - Connect output of "Get customers" to this node.  
   - Configure to keep only two fields renamed:  
     - `customerName` = `name`  
     - `customerId` = `id`  
   - Enable "Keep Only Set" to discard other fields.

7. **Create Merge node "Add customer name to charge data"**  
   - Connect output 1 from "Split array of charges to items" (charges)  
   - Connect output 2 from "Rename fields and keep only needed fields" (customers)  
   - Set mode to "Merge By Key"  
   - Property Name 1: `customer` (field in charges)  
   - Property Name 2: `customerId` (field in customers)  
   - This enriches charges with customer names.

8. **Create Pipedrive node "Search organisation"**  
   - Connect output 1 of "Add customer name to charge data" to this node.  
   - Resource: Organization  
   - Operation: Search  
   - Search term: `{{$json["customerName"]}}`  
   - Use Pipedrive OAuth credentials.

9. **Create Merge node "Add organisation Information to charge data"**  
   - Connect output 1 from "Add customer name to charge data"  
   - Connect output 2 from "Search organisation"  
   - Mode: Inner Join  
   - Merge by Index  
   - This attaches organization data to charges.

10. **Create Pipedrive node "Create note with charge information"**  
    - Connect output from "Add organisation Information to charge data"  
    - Resource: Note  
    - Operation: Create  
    - Content: `{{$json["description"]}}: {{$json["amount"] / 100}} {{$json["currency"]}}`  
    - Additional Fields: `org_id` = `{{$json["id"]}}` (organization ID)  
    - Use Pipedrive OAuth credentials.

11. **Create FunctionItem node "Set new last execution timestamp"**  
    - Connect output of "Create note with charge information" node.  
    - Code: Update global static data `lastExecution` with the timestamp from "Get last execution timestamp" node.  
    - Enable `Execute Once` to run only once per workflow run.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Stripe and Pipedrive credentials must be set up in n8n prior to running this workflow.                     | [Stripe credentials setup](https://docs.n8n.io/integrations/builtin/credentials/stripe/)                      |
|                                                                                                           | [Pipedrive credentials setup](https://docs.n8n.io/integrations/builtin/credentials/pipedrive/)                |
| The workflow depends on static data persistence in n8n for incremental queries. Ensure n8n static data is stable. |                                                                                                              |
| Pipedrive organization search uses customer names, which may not be unique; consider enhancing matching logic if needed. |                                                                                                              |
| Stripe API rate limits or large data volumes may impact workflow run times or cause failures.              |                                                                                                              |
| For more info on n8n function nodes and static data usage: https://docs.n8n.io/nodes/n8n-nodes-base.functionItem |                                                                                                              |