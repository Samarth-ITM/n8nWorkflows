Sync Replicated support bundles into Snowflake on a schedule

https://n8nworkflows.xyz/workflows/sync-replicated-support-bundles-into-snowflake-on-a-schedule-14519


# Sync Replicated support bundles into Snowflake on a schedule

# 1. Workflow Overview

This workflow synchronizes **Replicated support bundle metadata** into a **Snowflake table** on a scheduled basis. Its primary purpose is to maintain a refreshed reporting table of support bundles by clearing the target table, fetching the latest bundle list from the Replicated API, transforming the returned fields, and inserting the results into Snowflake.

Typical use cases include:

- Building internal reporting on support bundle activity
- Auditing uploaded support bundles over time
- Feeding BI dashboards from Snowflake
- Keeping a simple replicated snapshot of support bundle metadata

The workflow is organized into the following logical blocks.

## 1.1 Scheduled Initialization

The workflow begins with a weekly schedule trigger. Once triggered, it immediately truncates the destination Snowflake table so that the next load behaves like a full refresh.

## 1.2 Remote Data Retrieval

After the table is cleared, the workflow calls the Replicated API endpoint for support bundles. The API response is expected to contain a top-level `bundles` array.

## 1.3 Item Splitting and Field Mapping

The `bundles` array is split into one item per support bundle. The workflow then maps source API fields into an internal naming convention and prepares them again in the exact uppercase column naming expected by the Snowflake insert node.

## 1.4 Snowflake Load

Each mapped support bundle record is inserted into the Snowflake table `COMMON.REPLICATED_SUPPORTBUNDLES`.

## 1.5 Optional Table Preparation

A separate Snowflake node is included to create or recreate the destination table structure. This node is not connected to the main execution path, so it is intended for manual setup or schema reset.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Initialization

### Overview

This block starts the workflow on a schedule and resets the destination table before new data is loaded. It implements a **full refresh** strategy rather than an incremental sync.

### Nodes Involved

- Weekly Schedule Trigger
- Purge Support Bundles Table

### Node Details

#### Weekly Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point node that launches the workflow automatically based on a time rule.
- **Configuration choices:**
  - Configured with an interval rule
  - `triggerAtHour: 19`
  - The workflow description says “weekly,” but the JSON only shows an interval hour setting; the exact day/frequency should be verified in the n8n UI after import.
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input, as it is a trigger node
  - Outputs to `Purge Support Bundles Table`
- **Version-specific requirements:**
  - Uses `typeVersion: 1.1`
  - Behavior may vary slightly depending on n8n version and schedule UI interpretation
- **Edge cases or potential failure types:**
  - Timezone misunderstandings between server time and expected business time
  - Imported schedule rules may need validation after migration between n8n instances
  - Workflow must be activated for automatic execution
- **Sub-workflow reference:** None

#### Purge Support Bundles Table

- **Type and technical role:** `n8n-nodes-base.snowflake`  
  Executes SQL directly against Snowflake to remove all rows from the target table.
- **Configuration choices:**
  - Operation: `executeQuery`
  - Query: `TRUNCATE TABLE COMMON.REPLICATED_SUPPORTBUNDLES;`
  - `executeOnce: true`, which is appropriate here because only one truncate should happen per workflow run
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Weekly Schedule Trigger`
  - Output to `Fetch Replicated Support Bundles`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires valid Snowflake credentials configured in n8n
- **Edge cases or potential failure types:**
  - Snowflake authentication failure
  - Insufficient privileges for `TRUNCATE TABLE`
  - Target table missing
  - Schema/database context mismatch if credentials default to another database/schema
- **Sub-workflow reference:** None

---

## 2.2 Remote Data Retrieval

### Overview

This block retrieves support bundle data from Replicated using an authenticated HTTP request. It assumes the API returns a JSON payload containing a `bundles` array.

### Nodes Involved

- Fetch Replicated Support Bundles

### Node Details

#### Fetch Replicated Support Bundles

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Replicated API endpoint to fetch support bundle records.
- **Configuration choices:**
  - Method is not explicitly shown, so the node will use its default HTTP method, typically `GET`
  - URL: `https://api.replicated.com/vendor/v3/supportbundles`
  - Authentication: `genericCredentialType`
  - Generic auth type: `httpHeaderAuth`
  - `retryOnFail: true`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Purge Support Bundles Table`
  - Output to `Split Support Bundles`
- **Version-specific requirements:**
  - Uses `typeVersion: 4.1`
  - Requires an HTTP Header Auth credential in n8n, likely carrying the Replicated API token in a header such as `Authorization`
- **Edge cases or potential failure types:**
  - Authentication/header misconfiguration
  - API rate limiting
  - Non-200 response with partial or empty output because `alwaysOutputData` is enabled
  - Unexpected payload shape, especially if `bundles` is absent or renamed
  - Network timeout or TLS issues
- **Sub-workflow reference:** None

---

## 2.3 Item Splitting and Field Mapping

### Overview

This block converts the returned `bundles` array into individual items and maps the source properties into a normalized structure. It then remaps those normalized fields into the exact uppercase naming used by the Snowflake insert node.

### Nodes Involved

- Split Support Bundles
- Build Support Bundle Fields
- Set Support Bundle Attributes

### Node Details

#### Split Support Bundles

- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits an array field from a single input item into multiple output items.
- **Configuration choices:**
  - `fieldToSplitOut: bundles`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Fetch Replicated Support Bundles`
  - Output to `Build Support Bundle Fields`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - If the API response does not contain `bundles`, the node may output no items or fail depending on runtime behavior
  - If `bundles` is not an array, execution will not behave as intended
  - Empty array leads to no insertions downstream
- **Sub-workflow reference:** None

#### Build Support Bundle Fields

- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a normalized support bundle object from each split API item.
- **Configuration choices:**
  - `include: none`, meaning only the explicitly defined fields are preserved
  - Maps:
    - `supportbundle_id = {{$json.id}}`
    - `supportbundle_name = {{$json.name}}`
    - `supportbundle_uploaded_date = {{$json.uploadedAt}}`
    - `supportbundle_created_date = {{$json.collectedAt}}`
    - `supportbundle_source = {{$json.source}}`
    - `supportbundle_customer_id = {{$json.customerId}}`
    - `supportbundle_instance_id = {{$json.instanceId}}`
    - `supportbundle_size = {{$json.size}}` as number
    - `supportbundle_viewed = {{$json.viewed}}` as boolean
- **Key expressions or variables used:**
  - `$json.id`
  - `$json.name`
  - `$json.uploadedAt`
  - `$json.collectedAt`
  - `$json.source`
  - `$json.customerId`
  - `$json.instanceId`
  - `$json.size`
  - `$json.viewed`
- **Input and output connections:**
  - Input from `Split Support Bundles`
  - Output to `Set Support Bundle Attributes`
- **Version-specific requirements:**
  - Uses `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - Missing fields will produce null/empty values depending on node behavior
  - Type coercion issues if `size` is not numeric or `viewed` is not boolean
  - Date strings must be acceptable for downstream Snowflake timestamp conversion
- **Sub-workflow reference:** None

#### Set Support Bundle Attributes

- **Type and technical role:** `n8n-nodes-base.set`  
  Renames the normalized fields into uppercase column-aligned names expected by the Snowflake insert node.
- **Configuration choices:**
  - `include: none`
  - Maps:
    - `SUPPORTBUNDLE_ID = {{$json.supportbundle_id}}`
    - `SUPPORTBUNDLE_NAME = {{$json.supportbundle_name}}`
    - `SUPPORTBUNDLE_UPLOADED_DATE = {{$json.supportbundle_uploaded_date}}`
    - `SUPPORTBUNDLE_CREATED_DATE = {{$json.supportbundle_created_date}}`
    - `SUPPORTBUNDLE_SOURCE = {{$json.supportbundle_source}}`
    - `SUPPORTBUNDLE_CUSTOMER_ID = {{$json.supportbundle_customer_id}}`
    - `SUPPORTBUNDLE_INSTANCE_ID = {{$json.supportbundle_instance_id}}`
    - `SUPPORTBUNDLE_SIZE = {{$json.supportbundle_size}}` as number
    - `SUPPORTBUNDLE_VIEWED = {{$json.supportbundle_viewed}}` as boolean
- **Key expressions or variables used:**
  - `$json.supportbundle_id`
  - `$json.supportbundle_name`
  - `$json.supportbundle_uploaded_date`
  - `$json.supportbundle_created_date`
  - `$json.supportbundle_source`
  - `$json.supportbundle_customer_id`
  - `$json.supportbundle_instance_id`
  - `$json.supportbundle_size`
  - `$json.supportbundle_viewed`
- **Input and output connections:**
  - Input from `Build Support Bundle Fields`
  - Output to `Update Replicated Bundles`
- **Version-specific requirements:**
  - Uses `typeVersion: 3.2`
- **Edge cases or potential failure types:**
  - Mostly the same as the previous Set node
  - This node is somewhat redundant functionally, but helps align data shape exactly with Snowflake column names
- **Sub-workflow reference:** None

---

## 2.4 Snowflake Load

### Overview

This block inserts the processed support bundle items into Snowflake. Each output item from the previous mapping block becomes one row in the destination table.

### Nodes Involved

- Update Replicated Bundles

### Node Details

#### Update Replicated Bundles

- **Type and technical role:** `n8n-nodes-base.snowflake`  
  Inserts item data into the Snowflake table.
- **Configuration choices:**
  - Target table: `COMMON.REPLICATED_SUPPORTBUNDLES`
  - Columns:
    - `SUPPORTBUNDLE_ID`
    - `SUPPORTBUNDLE_NAME`
    - `SUPPORTBUNDLE_UPLOADED_DATE`
    - `SUPPORTBUNDLE_CREATED_DATE`
    - `SUPPORTBUNDLE_SOURCE`
    - `SUPPORTBUNDLE_CUSTOMER_ID`
    - `SUPPORTBUNDLE_INSTANCE_ID`
    - `SUPPORTBUNDLE_SIZE`
    - `SUPPORTBUNDLE_VIEWED`
  - The node is configured as a table-operation insert/update style node rather than raw SQL
- **Key expressions or variables used:**  
  Indirectly relies on the incoming item fields having names that exactly match the listed columns.
- **Input and output connections:**
  - Input from `Set Support Bundle Attributes`
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
  - Requires valid Snowflake credentials
- **Edge cases or potential failure types:**
  - Snowflake auth/role/warehouse issues
  - Insert failure if table does not exist
  - Timestamp parsing issues if incoming date strings are malformed
  - Null/empty values may fail if schema constraints are tightened beyond current definition
  - Duplicate handling is not implemented; because the table is truncated first, this is acceptable for a full refresh
- **Sub-workflow reference:** None

---

## 2.5 Optional Table Preparation

### Overview

This block contains a manual Snowflake setup node to create or recreate the destination table. It is not connected to the trigger path, so it must be run manually if needed.

### Nodes Involved

- Create Support Bundles Table

### Node Details

#### Create Support Bundles Table

- **Type and technical role:** `n8n-nodes-base.snowflake`  
  Executes a DDL statement to create or replace the target table schema.
- **Configuration choices:**
  - Operation: `executeQuery`
  - Query creates `COMMON.REPLICATED_SUPPORTBUNDLES` with columns:
    - `supportbundle_id STRING NOT NULL`
    - `supportbundle_name STRING`
    - `supportbundle_uploaded_date TIMESTAMP_NTZ`
    - `supportbundle_created_date TIMESTAMP_NTZ`
    - `supportbundle_source STRING`
    - `supportbundle_customer_id STRING`
    - `supportbundle_instance_id STRING`
    - `supportbundle_size NUMBER`
    - `supportbundle_viewed BOOLEAN`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input connection
  - No output connection
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Running `CREATE OR REPLACE TABLE` will drop and recreate the table, removing data
  - Requires Snowflake permissions for DDL
  - If the main workflow runs before this table exists, the truncate and insert steps will fail
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Schedule Trigger | Schedule Trigger | Starts the workflow on a time-based schedule |  | Purge Support Bundles Table | ## Initialize weekly process<br>Starts the weekly data flow by triggering and resetting the Snowflake table. |
| Purge Support Bundles Table | Snowflake | Truncates the destination Snowflake table before reload | Weekly Schedule Trigger | Fetch Replicated Support Bundles | ## Initialize weekly process<br>Starts the weekly data flow by triggering and resetting the Snowflake table. |
| Fetch Replicated Support Bundles | HTTP Request | Retrieves support bundle data from the Replicated API | Purge Support Bundles Table | Split Support Bundles | ## Retrieve and split data<br>Fetches support bundles via HTTP request and splits the data for processing. |
| Split Support Bundles | Split Out | Splits the `bundles` array into one item per support bundle | Fetch Replicated Support Bundles | Build Support Bundle Fields | ## Retrieve and split data<br>Fetches support bundles via HTTP request and splits the data for processing. |
| Build Support Bundle Fields | Set | Maps API fields into normalized internal bundle fields | Split Support Bundles | Set Support Bundle Attributes | ## Map and update data<br>Maps fields and updates the processed data in Snowflake. |
| Set Support Bundle Attributes | Set | Renames normalized fields to Snowflake-aligned uppercase column names | Build Support Bundle Fields | Update Replicated Bundles | ## Map and update data<br>Maps fields and updates the processed data in Snowflake. |
| Update Replicated Bundles | Snowflake | Inserts processed items into `COMMON.REPLICATED_SUPPORTBUNDLES` | Set Support Bundle Attributes |  | ## Map and update data<br>Maps fields and updates the processed data in Snowflake. |
| Create Support Bundles Table | Snowflake | Manually creates or replaces the destination Snowflake table |  |  | ## Prepare Snowflake table<br>Prepares the Snowflake database by creating necessary tables for support bundles. |
| Sticky Note | Sticky Note | Documentation panel for workflow purpose and setup |  |  | ## Replicated Support Bundles → Snowflake<br><br>### How it works<br><br>1. The workflow starts with a weekly schedule trigger.<br>2. It clears the Snowflake table for support bundles.<br>3. Support bundles data is retrieved via an HTTP request.<br>4. The retrieved data is split and manually mapped.<br>5. The mapped data is updated in the Snowflake database.<br><br>### Setup steps<br><br>- [ ] Ensure Snowflake database credentials are correctly set up.<br>- [ ] Ensure Replicated API credentials are configured for the HTTP request.<br>- [ ] Schedule the trigger for the desired interval.<br><br>### Customization<br><br>You can adjust the scheduling frequency to match your data processing needs. |
| Sticky Note1 | Sticky Note | Visual documentation for initialization block |  |  | ## Initialize weekly process<br><br>Starts the weekly data flow by triggering and resetting the Snowflake table. |
| Sticky Note2 | Sticky Note | Visual documentation for retrieval block |  |  | ## Retrieve and split data<br><br>Fetches support bundles via HTTP request and splits the data for processing. |
| Sticky Note3 | Sticky Note | Visual documentation for mapping/load block |  |  | ## Map and update data<br><br>Maps fields and updates the processed data in Snowflake. |
| Sticky Note4 | Sticky Note | Visual documentation for manual table setup block |  |  | ## Prepare Snowflake table<br><br>Prepares the Snowflake database by creating necessary tables for support bundles. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Replicated Support Bundles → Snowflake`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name it: `Weekly Schedule Trigger`
   - Configure it to run at the desired weekly time.
   - In the provided workflow, the visible configuration includes `triggerAtHour: 19`.
   - After creation, verify the exact recurrence rule in the UI because the exported JSON does not fully expose the intended weekly/day settings.

3. **Add a Snowflake node to clear the destination table**
   - Node type: **Snowflake**
   - Name it: `Purge Support Bundles Table`
   - Operation: **Execute Query**
   - SQL query:
     ```sql
     TRUNCATE TABLE COMMON.REPLICATED_SUPPORTBUNDLES;
     ```
   - Enable execution once per run if the option is available.
   - Configure Snowflake credentials:
     - Account
     - User
     - Password or key-based auth, depending on your n8n setup
     - Warehouse
     - Database
     - Schema
     - Role if required

4. **Connect the trigger to the purge node**
   - `Weekly Schedule Trigger` → `Purge Support Bundles Table`

5. **Add an HTTP Request node for the Replicated API**
   - Node type: **HTTP Request**
   - Name it: `Fetch Replicated Support Bundles`
   - Method: **GET**
   - URL:
     `https://api.replicated.com/vendor/v3/supportbundles`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Header Auth**
   - Enable:
     - `Retry On Fail`
     - `Always Output Data`
   - Create an HTTP Header Auth credential for Replicated:
     - Commonly this means adding the required authorization header expected by Replicated
     - Confirm the exact header name and token format from Replicated’s API documentation

6. **Connect purge to HTTP retrieval**
   - `Purge Support Bundles Table` → `Fetch Replicated Support Bundles`

7. **Add a Split Out node**
   - Node type: **Split Out**
   - Name it: `Split Support Bundles`
   - Field to split out: `bundles`
   - This assumes the API returns a JSON object containing an array named `bundles`.

8. **Connect HTTP retrieval to split**
   - `Fetch Replicated Support Bundles` → `Split Support Bundles`

9. **Add a Set node to normalize fields**
   - Node type: **Set**
   - Name it: `Build Support Bundle Fields`
   - Choose to keep **only** explicitly set fields
   - Add these fields:
     - `supportbundle_id` = `{{$json.id}}`
     - `supportbundle_name` = `{{$json.name}}`
     - `supportbundle_uploaded_date` = `{{$json.uploadedAt}}`
     - `supportbundle_created_date` = `{{$json.collectedAt}}`
     - `supportbundle_source` = `{{$json.source}}`
     - `supportbundle_customer_id` = `{{$json.customerId}}`
     - `supportbundle_instance_id` = `{{$json.instanceId}}`
     - `supportbundle_size` as **Number** = `{{$json.size}}`
     - `supportbundle_viewed` as **Boolean** = `{{$json.viewed}}`

10. **Connect split to normalized mapping**
    - `Split Support Bundles` → `Build Support Bundle Fields`

11. **Add a second Set node for Snowflake-aligned column names**
    - Node type: **Set**
    - Name it: `Set Support Bundle Attributes`
    - Keep **only** explicitly set fields
    - Add these fields:
      - `SUPPORTBUNDLE_ID` = `{{$json.supportbundle_id}}`
      - `SUPPORTBUNDLE_NAME` = `{{$json.supportbundle_name}}`
      - `SUPPORTBUNDLE_UPLOADED_DATE` = `{{$json.supportbundle_uploaded_date}}`
      - `SUPPORTBUNDLE_CREATED_DATE` = `{{$json.supportbundle_created_date}}`
      - `SUPPORTBUNDLE_SOURCE` = `{{$json.supportbundle_source}}`
      - `SUPPORTBUNDLE_CUSTOMER_ID` = `{{$json.supportbundle_customer_id}}`
      - `SUPPORTBUNDLE_INSTANCE_ID` = `{{$json.supportbundle_instance_id}}`
      - `SUPPORTBUNDLE_SIZE` as **Number** = `{{$json.supportbundle_size}}`
      - `SUPPORTBUNDLE_VIEWED` as **Boolean** = `{{$json.supportbundle_viewed}}`

12. **Connect the two Set nodes**
    - `Build Support Bundle Fields` → `Set Support Bundle Attributes`

13. **Add a Snowflake node to insert the records**
    - Node type: **Snowflake**
    - Name it: `Update Replicated Bundles`
    - Configure it to insert rows into a table
    - Table:
      `COMMON.REPLICATED_SUPPORTBUNDLES`
    - Columns:
      - `SUPPORTBUNDLE_ID`
      - `SUPPORTBUNDLE_NAME`
      - `SUPPORTBUNDLE_UPLOADED_DATE`
      - `SUPPORTBUNDLE_CREATED_DATE`
      - `SUPPORTBUNDLE_SOURCE`
      - `SUPPORTBUNDLE_CUSTOMER_ID`
      - `SUPPORTBUNDLE_INSTANCE_ID`
      - `SUPPORTBUNDLE_SIZE`
      - `SUPPORTBUNDLE_VIEWED`
    - Use the same Snowflake credentials as the purge node

14. **Connect the final Set node to the insert node**
    - `Set Support Bundle Attributes` → `Update Replicated Bundles`

15. **Add a manual setup node to create the table**
    - Node type: **Snowflake**
    - Name it: `Create Support Bundles Table`
    - Operation: **Execute Query**
    - SQL:
      ```sql
      CREATE OR REPLACE TABLE COMMON.REPLICATED_SUPPORTBUNDLES (
          supportbundle_id STRING NOT NULL,
          supportbundle_name STRING,
          supportbundle_uploaded_date TIMESTAMP_NTZ,
          supportbundle_created_date TIMESTAMP_NTZ,
          supportbundle_source STRING,
          supportbundle_customer_id STRING,
          supportbundle_instance_id STRING,
          supportbundle_size NUMBER,
          supportbundle_viewed BOOLEAN
      );
      ```
    - Leave it unconnected if you want to keep it as a manual maintenance/setup node.
    - Run this node once before enabling the main workflow if the table does not yet exist.

16. **Optionally add documentation sticky notes**
    - Add one general note with purpose, setup reminders, and customization notes.
    - Add separate notes for:
      - Initialization
      - Retrieval and splitting
      - Mapping and updating
      - Snowflake table preparation

17. **Test the workflow manually**
    - First run `Create Support Bundles Table` manually if needed.
    - Then execute the main path manually from the trigger or from `Purge Support Bundles Table`.
    - Confirm:
      - The API returns a `bundles` array
      - Items are split correctly
      - Set nodes produce expected field names and types
      - Snowflake receives rows successfully

18. **Validate the schedule and activate**
    - Confirm the recurrence settings in the trigger.
    - Activate the workflow so it can run automatically.

## Credential Setup Requirements

### Snowflake credentials
You will need:
- Snowflake account identifier
- Username
- Password or supported authentication method
- Warehouse
- Database
- Schema
- Optional role with permission to:
  - `TRUNCATE TABLE`
  - `CREATE OR REPLACE TABLE`
  - `INSERT` into `COMMON.REPLICATED_SUPPORTBUNDLES`

### Replicated API credentials
You will need:
- An **HTTP Header Auth** credential in n8n
- The correct header name and value expected by Replicated’s API
- Access rights to the endpoint:
  `https://api.replicated.com/vendor/v3/supportbundles`

## Input/Output Expectations

### API response expectation
The HTTP Request node should return a JSON object containing:
- `bundles`: array of support bundle objects

Each bundle is expected to include fields such as:
- `id`
- `name`
- `uploadedAt`
- `collectedAt`
- `source`
- `customerId`
- `instanceId`
- `size`
- `viewed`

### Snowflake output expectation
The final table stores:
- bundle ID
- name
- uploaded timestamp
- created/collected timestamp
- source
- customer ID
- instance ID
- size
- viewed flag

## Important Rebuild Notes

- The workflow performs a **destructive full refresh** by truncating the table first.
- The `Create Support Bundles Table` node is not part of the scheduled chain.
- If the API schema changes, the Set expressions will need updating.
- If you want historical retention instead of snapshot refresh, replace the truncate strategy with merge/upsert logic.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replicated Support Bundles → Snowflake | Workflow title / overall purpose |
| The process starts on a schedule, clears the target table, fetches support bundle data, splits the bundle array, maps fields, and writes the result into Snowflake. | Functional summary |
| Ensure Snowflake database credentials are correctly set up. | Deployment/setup note |
| Ensure Replicated API credentials are configured for the HTTP request. | Deployment/setup note |
| Schedule the trigger for the desired interval. | Deployment/setup note |
| You can adjust the scheduling frequency to match your data processing needs. | Customization note |
| The workflow includes a separate manual node to create or recreate the Snowflake table before scheduled use. | Operational note |