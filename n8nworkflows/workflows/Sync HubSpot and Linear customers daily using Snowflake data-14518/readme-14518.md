Sync HubSpot and Linear customers daily using Snowflake data

https://n8nworkflows.xyz/workflows/sync-hubspot-and-linear-customers-daily-using-snowflake-data-14518


# Sync HubSpot and Linear customers daily using Snowflake data

# 1. Workflow Overview

This workflow synchronizes customer account data from HubSpot-derived Snowflake tables into Linear on a daily schedule. It retrieves closed-won company data from Snowflake, fetches the full customer list from Linear with pagination, compares both datasets, then creates or updates Linear customers accordingly. A Slack message is sent for each planned action.

Primary use cases:
- Keep Linear customer records aligned with revenue and seat data from HubSpot/Snowflake
- Create missing customers in Linear
- Update existing Linear customers when ARR or seat count changes
- Notify a Slack channel about sync actions performed

## 1.1 Scheduled Trigger and Source Data Retrieval
The workflow starts on a daily schedule at 2 PM, queries Snowflake for HubSpot company/deal aggregates, and optionally passes through a disabled limiting node intended for testing.

## 1.2 Linear Customer Pagination
The workflow initializes pagination state, repeatedly calls the Linear GraphQL API for customers, processes each page, and loops until no further pages exist.

## 1.3 Final Linear Dataset Formatting
Once pagination completes, the collected customer list is reshaped into a normalized structure for downstream comparison logic.

## 1.4 Matching and Action Decision
Snowflake customer records are compared against Linear customers using external IDs first and domains as fallback. Each Snowflake company is classified as either:
- `create` when no Linear match is found
- `update` when a Linear match exists and revenue or seat count has changed

## 1.5 Linear Customer Write Operations
The workflow routes each action to the appropriate Linear GraphQL mutation:
- create customer
- update customer

## 1.6 Slack Notification
In parallel with create/update routing, each planned action generates a Slack notification summarizing the intended sync result.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Initial Data Fetch

**Overview:**  
This block starts the workflow on a fixed schedule and retrieves the authoritative customer dataset from Snowflake. It defines the source records that will later be matched against Linear customers.

**Nodes Involved:**  
- When Scheduled at 2 PM
- Retrieve HubSpot Data from Snowflake
- Limit HubSpot Records

### Node: When Scheduled at 2 PM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point
- **Configuration choices:** Configured to trigger daily at hour 14 (2 PM)
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none
  - Output: `Retrieve HubSpot Data from Snowflake`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or potential failure types:**
  - Misconfigured timezone at workflow or instance level may cause unexpected runtime
  - If the workflow is inactive, it will never trigger
- **Sub-workflow reference:** None

### Node: Retrieve HubSpot Data from Snowflake
- **Type and technical role:** `n8n-nodes-base.snowflake`; executes SQL against Snowflake
- **Configuration choices:** Runs a query joining HubSpot companies, associations, and deals tables. It filters for `closedwon` deals, aggregates ARR with `SUM(d.AMOUNT)`, computes seats with `MAX(d.NUMBER_OF_SEATS)`, groups by company, and orders descending by ARR.
- **Key expressions or variables used:** None; static SQL query
- **Input and output connections:**
  - Input: `When Scheduled at 2 PM`
  - Output: `Limit HubSpot Records`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Snowflake authentication or network failures
  - Table/schema name mismatches
  - Null `DOMAIN`, `AMOUNT`, or `NUMBER_OF_SEATS` values affecting downstream logic
  - Duplicate or unexpected company rows if source associations are inconsistent
- **Sub-workflow reference:** None

### Node: Limit HubSpot Records
- **Type and technical role:** `n8n-nodes-base.limit`; optional test limiter
- **Configuration choices:** Limits items to 10, but the node is disabled
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Retrieve HubSpot Data from Snowflake`
  - Output: `Set Initial Pagination`
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - Important implementation detail: because the node is disabled, n8n passes data through it transparently
  - Downstream code still references this node by name, so renaming it can break expressions/code even while disabled
- **Sub-workflow reference:** None

---

## 2.2 Pagination Setup and Iteration

**Overview:**  
This block retrieves all Linear customers page by page through the GraphQL API. It keeps state across iterations using cursor-based pagination until `hasNextPage` becomes false.

**Nodes Involved:**  
- Set Initial Pagination
- Fetch Customer Data
- Process Customer Page
- If More Pages Exist
- Set Next Page Parameters

### Node: Set Initial Pagination
- **Type and technical role:** `n8n-nodes-base.set`; initializes pagination state
- **Configuration choices:** Sets:
  - `cursor` = empty string
  - `pageNumber` = `0`
  - `allCustomers` = empty array
- **Key expressions or variables used:** Literal initialization values
- **Input and output connections:**
  - Input: `Limit HubSpot Records`
  - Output: `Fetch Customer Data`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If upstream returns zero items, downstream pagination may also execute zero times depending on execution context
- **Sub-workflow reference:** None

### Node: Fetch Customer Data
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls Linear GraphQL API
- **Configuration choices:** Sends a POST to `https://api.linear.app/graphql` using predefined Linear credentials. Query requests up to 250 customers per page with fields:
  - `id`, `name`, `domains`, `externalIds`, `revenue`, `size`
  - `tier`, `status`, `owner`, `logoUrl`, `createdAt`, `updatedAt`
  - `pageInfo { hasNextPage endCursor }`
- **Key expressions or variables used:**
  - GraphQL variables body is dynamically built:
    - if `cursor` exists: `{"after": "<cursor>"}`
    - otherwise: `{}`
- **Input and output connections:**
  - Inputs: `Set Initial Pagination`, `Set Next Page Parameters`
  - Output: `Process Customer Page`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**
  - Linear authentication failure
  - GraphQL query errors returned in body
  - Invalid cursor causing paging failure
  - API rate limiting if customer volume is large or workflow is frequently run
  - Large customer lists may increase memory usage in later accumulation
- **Sub-workflow reference:** None

### Node: Process Customer Page
- **Type and technical role:** `n8n-nodes-base.code`; accumulates results across pages
- **Configuration choices:** Reads the current API response, extracts `nodes` and `pageInfo`, retrieves previous accumulated state either from:
  - `Set Next Page Parameters` for subsequent pages, or
  - `Set Initial Pagination` for the first page  
  Then appends current page customers to `allCustomers`, increments `pageNumber`, and emits updated pagination state.
- **Key expressions or variables used:**
  - `$('Set Next Page Parameters').all()[0].json`
  - fallback to `$('Set Initial Pagination').all()[0].json`
- **Input and output connections:**
  - Input: `Fetch Customer Data`
  - Output: `If More Pages Exist`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If Linear response shape changes, references like `response.data.customers.nodes` will fail
  - If neither pagination state node is available due to structural changes, the code throws
  - Large arrays may cause memory pressure
  - Console logs are useful for execution logs but not surfaced as data
- **Sub-workflow reference:** None

### Node: If More Pages Exist
- **Type and technical role:** `n8n-nodes-base.if`; loop control
- **Configuration choices:** Checks whether `hasNextPage` is `true`
- **Key expressions or variables used:**
  - `={{ $json.hasNextPage }}`
- **Input and output connections:**
  - Input: `Process Customer Page`
  - True output: `Set Next Page Parameters`
  - False output: `Format Output Data`
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - If `hasNextPage` is missing or not boolean, strict validation may fail or branch unexpectedly
- **Sub-workflow reference:** None

### Node: Set Next Page Parameters
- **Type and technical role:** `n8n-nodes-base.set`; persists loop state for next request
- **Configuration choices:** Carries forward:
  - `cursor`
  - `pageNumber`
  - `allCustomers`
- **Key expressions or variables used:**
  - `={{ $json.cursor }}`
  - `={{ $json.pageNumber }}`
  - `={{ $json.allCustomers }}`
- **Input and output connections:**
  - Input: `If More Pages Exist` (true branch)
  - Output: `Fetch Customer Data`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If previous node outputs malformed state, pagination loop can stall or fail
- **Sub-workflow reference:** None

---

## 2.3 Data Processing and Formatting

**Overview:**  
After pagination completes, this block converts the accumulated Linear customers into a predictable output structure. This normalized structure mirrors GraphQL-style nesting expected by the matching code.

**Nodes Involved:**  
- Format Output Data

### Node: Format Output Data
- **Type and technical role:** `n8n-nodes-base.code`; final data reshaping
- **Configuration choices:** Takes the first input item, extracts `allCustomers`, logs the total count, and returns:
  - `data.customers.nodes = allCustomers`
- **Key expressions or variables used:**
  - `const data = $input.all()[0].json;`
- **Input and output connections:**
  - Input: `If More Pages Exist` (false branch)
  - Output: `Match and Update Customers`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - If `allCustomers` is missing or not an array, downstream logic fails
  - Only the first input item is used; changes to upstream cardinality could affect correctness
- **Sub-workflow reference:** None

---

## 2.4 Customer Matching and Routing

**Overview:**  
This block compares Snowflake customers against all retrieved Linear customers. It determines whether each Snowflake company should be created in Linear or updated there, then routes each result to the correct downstream action.

**Nodes Involved:**  
- Match and Update Customers
- Route by Customer Action

### Node: Match and Update Customers
- **Type and technical role:** `n8n-nodes-base.code`; reconciliation and action generation
- **Configuration choices:**  
  The node:
  1. Reads Snowflake output from `Limit HubSpot Records`
  2. Normalizes fields into:
     - `companyId`
     - `name`
     - `domain`
     - `arr`
     - `seats`
  3. Reads Linear customers from `Format Output Data`
  4. Builds two lookup maps:
     - by every `externalId`
     - by every `domain`
  5. Matches each Snowflake customer by external ID first, then domain fallback
  6. Produces:
     - `create` items for unmatched customers
     - `update` items for matched customers where `revenue` or `size` changed
- **Key expressions or variables used:**
  - `$('Limit HubSpot Records').all()`
  - `$('Format Output Data').all()[0].json.data.customers.nodes`
- **Input and output connections:**
  - Input: `Format Output Data`
  - Outputs:
    - `Route by Customer Action`
    - `Post Slack Notification`
- **Version-specific requirements:** Type version `2`
- **Edge cases or potential failure types:**
  - Renaming `Limit HubSpot Records` breaks the code reference
  - Disabled pass-through behavior of the limit node is relied upon
  - Domain matching can create false positives if domains are not unique or normalized
  - `DOMAIN` null values reduce fallback matching quality
  - Differences in casing or formatting of domains/external IDs are not normalized
  - Existing Linear customers with multiple external IDs are handled correctly, but duplicate mappings may still create ambiguity
  - Only revenue and size changes trigger updates; name/domain/externalIds changes are ignored
- **Sub-workflow reference:** None

### Node: Route by Customer Action
- **Type and technical role:** `n8n-nodes-base.switch`; branches records by action type
- **Configuration choices:** Two named outputs:
  - `create` when `$json.action === "create"`
  - `update` when `$json.action === "update"`
- **Key expressions or variables used:**
  - `={{ $json.action }}`
- **Input and output connections:**
  - Input: `Match and Update Customers`
  - Create output: `Post Create to Linear`
  - Update output: `Post Update to Linear`
- **Version-specific requirements:** Type version `3.2`
- **Edge cases or potential failure types:**
  - Any item with unexpected `action` value is dropped from both branches
  - Strict type validation expects string values
- **Sub-workflow reference:** None

---

## 2.5 Customer Management in Linear

**Overview:**  
This block performs the actual write operations in Linear using GraphQL mutations. It either creates a new customer or updates revenue and seat values on an existing one.

**Nodes Involved:**  
- Post Create to Linear
- Post Update to Linear

### Node: Post Create to Linear
- **Type and technical role:** `n8n-nodes-base.httpRequest`; GraphQL mutation to create a customer
- **Configuration choices:** Sends POST request to Linear GraphQL API with mutation `customerCreate`. The input payload includes:
  - `name` from Snowflake company name
  - `domains` array containing the domain
  - `externalIds` array containing Snowflake company ID
  - `revenue`
  - `size`
- **Key expressions or variables used:**
  - `{{ $json.snowflakeData.name }}`
  - `{{ $json.snowflakeData.domain }}`
  - `{{ $json.snowflakeData.companyId }}`
  - `{{ $json.snowflakeData.arr }}`
  - `{{ $json.snowflakeData.seats }}`
- **Input and output connections:**
  - Input: `Route by Customer Action` (`create`)
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**
  - Linear authentication or permission failure
  - GraphQL validation errors if domain or values are invalid
  - Possible duplicate customer creation if matching logic misses an existing record
  - Null or malformed domain may be rejected
- **Sub-workflow reference:** None

### Node: Post Update to Linear
- **Type and technical role:** `n8n-nodes-base.httpRequest`; GraphQL mutation to update a customer
- **Configuration choices:** Sends POST request to Linear GraphQL API with mutation `customerUpdate`. It updates:
  - `revenue`
  - `size`
  for the customer identified by `linearId`
- **Key expressions or variables used:**
  - `{{ $json.linearId }}`
  - `{{ $json.snowflakeData.arr }}`
  - `{{ $json.snowflakeData.seats }}`
- **Input and output connections:**
  - Input: `Route by Customer Action` (`update`)
  - Output: none
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or potential failure types:**
  - Invalid or stale `linearId`
  - GraphQL mutation may partially fail or return `success: false`
  - Revenue/size type mismatches if upstream data is not numeric
- **Sub-workflow reference:** None

---

## 2.6 Slack Notification

**Overview:**  
This block posts a Slack message for each action item generated by the matching logic. It is configured not to stop the workflow if Slack delivery fails.

**Nodes Involved:**  
- Post Slack Notification

### Node: Post Slack Notification
- **Type and technical role:** `n8n-nodes-base.slack`; sends block-based Slack messages
- **Configuration choices:**  
  Sends a block message to a named channel (`#your-slack-channel`) with:
  - header: “🔄 Linear Customer Sync”
  - one field showing action, customer name, ARR, and seats  
  `onError` is set to continue regular output, and retries are disabled.
- **Key expressions or variables used:**
  - `{{ $json.action.toTitleCase() }}`
  - `{{ $json.snowflakeData.name }}`
  - `${{ $json.snowflakeData.arr }}`
  - `{{ $json.snowflakeData.seats }}`
- **Input and output connections:**
  - Input: `Match and Update Customers`
  - Output: none
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - Slack auth or channel access failure
  - Invalid channel name
  - Message formatting issues if fields are null
  - Since it runs in parallel to Linear writes, the notification reports intended action, not confirmed success of create/update
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Documentation note |  |  | ## HubSpot → Linear Customers Sync (via Snowflake)<br>### How it works<br>1. Scheduled trigger initiates data retrieval from HubSpot via Snowflake.<br>2. Data is processed and paginated results are fetched.<br>3. Final output is formatted and customers are matched.<br>4. Based on the match, the workflow routes to either create or update customers in Linear.<br>5. Slack update is sent with the results.<br>### Setup steps<br>- [ ] Configure Snowflake credentials for data retrieval.<br>- [ ] Set up Linear API credentials for customer management.<br>- [ ] Connect Slack account for updates.<br>- [ ] Schedule the trigger according to desired frequency. |
| Sticky Note1 | stickyNote | Documentation note |  |  | ## Trigger and initial data fetch<br>Initial trigger and data retrieval from HubSpot via Snowflake. |
| Sticky Note2 | stickyNote | Documentation note |  |  | ## Pagination setup and iteration<br>Set up pagination and iterate through pages of customer data. |
| Sticky Note3 | stickyNote | Documentation note |  |  | ## Data processing and formatting<br>Process fetched data and format the final output. |
| Sticky Note4 | stickyNote | Documentation note |  |  | ## Customer matching and routing<br>Match customers for actions and route to create or update functions. |
| Sticky Note5 | stickyNote | Documentation note |  |  | ## Customer management in Linear<br>Create or update customers in Linear based on the routing decision. |
| Sticky Note6 | stickyNote | Documentation note |  |  | ## Notify via Slack<br>Send a Slack update with the result of the customer management process. |
| When Scheduled at 2 PM | scheduleTrigger | Daily workflow trigger |  | Retrieve HubSpot Data from Snowflake | ## Trigger and initial data fetch<br>Initial trigger and data retrieval from HubSpot via Snowflake. |
| Retrieve HubSpot Data from Snowflake | snowflake | Query HubSpot-derived company metrics from Snowflake | When Scheduled at 2 PM | Limit HubSpot Records | ## Trigger and initial data fetch<br>Initial trigger and data retrieval from HubSpot via Snowflake. |
| Limit HubSpot Records | limit | Optional testing limiter; currently disabled pass-through | Retrieve HubSpot Data from Snowflake | Set Initial Pagination | ## Trigger and initial data fetch<br>Initial trigger and data retrieval from HubSpot via Snowflake. |
| Set Initial Pagination | set | Initialize cursor, page number, and customer accumulator | Limit HubSpot Records | Fetch Customer Data | ## Pagination setup and iteration<br>Set up pagination and iterate through pages of customer data. |
| Fetch Customer Data | httpRequest | Fetch Linear customers page by page via GraphQL | Set Initial Pagination; Set Next Page Parameters | Process Customer Page | ## Pagination setup and iteration<br>Set up pagination and iterate through pages of customer data. |
| Process Customer Page | code | Merge current Linear page into accumulated results | Fetch Customer Data | If More Pages Exist | ## Pagination setup and iteration<br>Set up pagination and iterate through pages of customer data. |
| If More Pages Exist | if | Decide whether to continue pagination loop | Process Customer Page | Set Next Page Parameters; Format Output Data | ## Pagination setup and iteration<br>Set up pagination and iterate through pages of customer data. |
| Set Next Page Parameters | set | Carry cursor and accumulated state into next iteration | If More Pages Exist | Fetch Customer Data | ## Pagination setup and iteration<br>Set up pagination and iterate through pages of customer data. |
| Format Output Data | code | Normalize final Linear dataset structure | If More Pages Exist | Match and Update Customers | ## Data processing and formatting<br>Process fetched data and format the final output. |
| Match and Update Customers | code | Compare Snowflake and Linear customers, generate create/update actions | Format Output Data | Route by Customer Action; Post Slack Notification | ## Customer matching and routing<br>Match customers for actions and route to create or update functions. |
| Route by Customer Action | switch | Branch action items into create vs update paths | Match and Update Customers | Post Create to Linear; Post Update to Linear | ## Customer matching and routing<br>Match customers for actions and route to create or update functions. |
| Post Create to Linear | httpRequest | Create new Linear customer | Route by Customer Action |  | ## Customer management in Linear<br>Create or update customers in Linear based on the routing decision. |
| Post Update to Linear | httpRequest | Update existing Linear customer revenue and size | Route by Customer Action |  | ## Customer management in Linear<br>Create or update customers in Linear based on the routing decision. |
| Post Slack Notification | slack | Send Slack summary per action item | Match and Update Customers |  | ## Notify via Slack<br>Send a Slack update with the result of the customer management process. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `HubSpot → Linear Customers Sync (via Snowflake)`

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `When Scheduled at 2 PM`
   - Configure one interval rule with trigger hour `14`
   - Confirm the instance/workflow timezone matches your intended schedule

3. **Add a Snowflake node**
   - Node type: `Snowflake`
   - Name: `Retrieve HubSpot Data from Snowflake`
   - Operation: `Execute Query`
   - Configure Snowflake credentials:
     - account
     - database
     - warehouse
     - username/password or supported auth method
     - role if needed
   - Use this SQL query:

     ```sql
     SELECT
       comp.COMPANY_ID,
       comp.NAME AS COMPANY_NAME,
       comp.DOMAIN,
       SUM(d.AMOUNT) AS ARR,
       MAX(d.NUMBER_OF_SEATS) AS SEATS
     FROM
       "HUBSPOT"."COMPANIES" comp
     JOIN
       "HUBSPOT"."ASSOCIATIONS_DEALS_TO_COMPANIES" assoc
       ON comp.COMPANY_ID = assoc.COMPANY_ID
     JOIN
       "HUBSPOT"."DEALS" d
       ON assoc.DEAL_ID = d.DEAL_ID
     WHERE
       d.DEALSTAGE = 'closedwon'
     GROUP BY
       comp.COMPANY_ID,
       comp.NAME,
       comp.DOMAIN
     ORDER BY
       ARR DESC
     ```

4. **Add a Limit node for optional testing**
   - Node type: `Limit`
   - Name: `Limit HubSpot Records`
   - Set max items to `10`
   - Disable the node after creating it
   - Important: keep this exact node name if you want the provided code to work unchanged

5. **Connect the first block**
   - `When Scheduled at 2 PM` → `Retrieve HubSpot Data from Snowflake`
   - `Retrieve HubSpot Data from Snowflake` → `Limit HubSpot Records`

6. **Add a Set node to initialize pagination**
   - Node type: `Set`
   - Name: `Set Initial Pagination`
   - Add fields:
     - `cursor` as string = empty string
     - `pageNumber` as number = `0`
     - `allCustomers` as array = empty array `[]`

7. **Add an HTTP Request node to fetch Linear customers**
   - Node type: `HTTP Request`
   - Name: `Fetch Customer Data`
   - Method: `POST`
   - URL: `https://api.linear.app/graphql`
   - Authentication: predefined credential type
   - Credential type: `Linear API`
   - Configure Linear credentials with a valid API key/token
   - Set body type to JSON
   - Use this body expression:

     ```json
     {
       "query": "query GetCustomers($after: String) { customers(first: 250, after: $after) { nodes { id name domains externalIds revenue size tier { id name } status { id name } owner { id name email } logoUrl createdAt updatedAt } pageInfo { hasNextPage endCursor } } }",
       "variables": {{ $json.cursor ? '{"after": "' + $json.cursor + '"}' : '{}' }}
     }
     ```

8. **Add a Code node to process each fetched page**
   - Node type: `Code`
   - Name: `Process Customer Page`
   - Language: JavaScript
   - Paste:

     ```javascript
     // Process page results and prepare for next iteration
     const response = $input.all()[0].json;
     const customers = response.data.customers.nodes;
     const pageInfo = response.data.customers.pageInfo;

     // Get accumulated data — use Prepare Next Iteration if available (pages 2+),
     // otherwise fall back to Initialize Pagination (first page)
     let previousData;
     try {
       previousData = $('Set Next Page Parameters').all()[0].json;
     } catch (e) {
       previousData = $('Set Initial Pagination').all()[0].json;
     }

     const allCustomers = [...(previousData.allCustomers || [])];
     const pageNumber = (previousData.pageNumber || 0) + 1;

     // Add new customers to collection
     allCustomers.push(...customers);

     console.log(`📄 Page ${pageNumber}: Fetched ${customers.length} customers (Total: ${allCustomers.length})`);

     return [{
       json: {
         cursor: pageInfo.endCursor,
         hasNextPage: pageInfo.hasNextPage,
         pageNumber: pageNumber,
         allCustomers: allCustomers,
         currentPageCustomers: customers
       }
     }];
     ```

9. **Add an If node for pagination continuation**
   - Node type: `If`
   - Name: `If More Pages Exist`
   - Condition:
     - left value: `={{ $json.hasNextPage }}`
     - operator: `is true`

10. **Add a Set node to carry next-page state**
    - Node type: `Set`
    - Name: `Set Next Page Parameters`
    - Add fields:
      - `cursor` = `={{ $json.cursor }}`
      - `pageNumber` = `={{ $json.pageNumber }}`
      - `allCustomers` = `={{ $json.allCustomers }}`

11. **Connect the pagination loop**
    - `Limit HubSpot Records` → `Set Initial Pagination`
    - `Set Initial Pagination` → `Fetch Customer Data`
    - `Fetch Customer Data` → `Process Customer Page`
    - `Process Customer Page` → `If More Pages Exist`
    - `If More Pages Exist` true branch → `Set Next Page Parameters`
    - `Set Next Page Parameters` → `Fetch Customer Data`

12. **Add a Code node to format final Linear output**
    - Node type: `Code`
    - Name: `Format Output Data`
    - Paste:

      ```javascript
      // Format final output to match expected structure
      const data = $input.all()[0].json;
      const allCustomers = data.allCustomers;

      console.log(`✅ Pagination complete: ${allCustomers.length} total customers`);

      return [{
        json: {
          data: {
            customers: {
              nodes: allCustomers
            }
          }
        }
      }];
      ```

13. **Connect pagination completion**
    - `If More Pages Exist` false branch → `Format Output Data`

14. **Add a Code node for matching and action generation**
    - Node type: `Code`
    - Name: `Match and Update Customers`
    - Paste:

      ```javascript
      // Customer matching: create and update only

      // Get Snowflake customers
      const snowflakeCustomers = $('Limit HubSpot Records').all().map(item => ({
        companyId: item.json.COMPANY_ID.toString(),
        name: item.json.COMPANY_NAME,
        domain: item.json.DOMAIN,
        arr: Math.round(item.json.ARR || 0),
        seats: item.json.SEATS || 0
      }));

      // Get Linear customers
      const linearData = $('Format Output Data').all()[0].json.data.customers.nodes;

      // Build lookup maps
      // 1. Index ALL externalIds (not just [0]) — handles multi-ID customers
      //    and cases where Snowflake ID matches a non-primary externalId
      const linearByExternalId = new Map();
      linearData.forEach(c => {
        (c.externalIds || []).forEach(id => {
          if (!linearByExternalId.has(id)) linearByExternalId.set(id, c);
        });
      });

      // 2. Domain-based fallback — handles cases where HubSpot re-keyed
      //    the company (new COMPANY_ID) but the domain is unchanged
      const linearByDomain = new Map();
      linearData.forEach(c => {
        (c.domains || []).forEach(d => {
          if (d && !linearByDomain.has(d)) linearByDomain.set(d, c);
        });
      });

      // Resolve a Snowflake customer to its Linear counterpart
      function findLinearCustomer(sfCustomer) {
        return linearByExternalId.get(sfCustomer.companyId)
          || linearByDomain.get(sfCustomer.domain)
          || null;
      }

      const toCreate = [];
      const toUpdate = [];
      const matchLog = [];

      snowflakeCustomers.forEach(sfCustomer => {
        const linearCustomer = findLinearCustomer(sfCustomer);

        if (!linearCustomer) {
          toCreate.push({ action: 'create', snowflakeData: sfCustomer });
          return;
        }

        const matchedVia = linearByExternalId.has(sfCustomer.companyId) ? 'externalId' : 'domain';
        matchLog.push(`  ✓ [${matchedVia}] ${sfCustomer.name}`);

        const revenueChanged = linearCustomer.revenue !== sfCustomer.arr;
        const sizeChanged = linearCustomer.size !== sfCustomer.seats;

        if (revenueChanged || sizeChanged) {
          toUpdate.push({
            action: 'update',
            linearId: linearCustomer.id,
            linearData: linearCustomer,
            snowflakeData: sfCustomer,
            changes: { revenue: revenueChanged, size: sizeChanged }
          });
        }
      });

      console.log('\n📈 Action Summary:');
      console.log('  🆕 Create: ' + toCreate.length);
      console.log('  ✏️ Update: ' + toUpdate.length);
      console.log('  📊 Total: ' + (toCreate.length + toUpdate.length));
      if (matchLog.length) {
        console.log('\n🔗 Domain fallback matches:');
        matchLog.filter(l => l.includes('domain')).forEach(l => console.log(l));
      }

      return [
        ...toCreate.map(item => ({ json: item })),
        ...toUpdate.map(item => ({ json: item }))
      ];
      ```

15. **Connect formatting to matching**
    - `Format Output Data` → `Match and Update Customers`

16. **Add a Switch node for action routing**
    - Node type: `Switch`
    - Name: `Route by Customer Action`
    - Create two outputs with renamed output keys:
      - `create` where `={{ $json.action }}` equals `create`
      - `update` where `={{ $json.action }}` equals `update`

17. **Add the create mutation HTTP node**
    - Node type: `HTTP Request`
    - Name: `Post Create to Linear`
    - Method: `POST`
    - URL: `https://api.linear.app/graphql`
    - Authentication: predefined credential type
    - Credential type: `Linear API`
    - JSON body:

      ```json
      {
        "query": "mutation CustomerCreate($input: CustomerCreateInput!) { customerCreate(input: $input) { success customer { id name domains externalIds revenue size status { id name } } } }",
        "variables": {
          "input": {
            "name": "{{ $json.snowflakeData.name }}",
            "domains": ["{{ $json.snowflakeData.domain }}"],
            "externalIds": ["{{ $json.snowflakeData.companyId }}"],
            "revenue": {{ $json.snowflakeData.arr }},
            "size": {{ $json.snowflakeData.seats }}
          }
        }
      }
      ```

18. **Add the update mutation HTTP node**
    - Node type: `HTTP Request`
    - Name: `Post Update to Linear`
    - Method: `POST`
    - URL: `https://api.linear.app/graphql`
    - Authentication: predefined credential type
    - Credential type: `Linear API`
    - JSON body:

      ```json
      {
        "query": "mutation CustomerUpdate($id: String!, $input: CustomerUpdateInput!) { customerUpdate(id: $id, input: $input) { success customer { id name revenue size status { id name } } } }",
        "variables": {
          "id": "{{ $json.linearId }}",
          "input": {
            "revenue": {{ $json.snowflakeData.arr }},
            "size": {{ $json.snowflakeData.seats }}
          }
        }
      }
      ```

19. **Connect routing**
    - `Match and Update Customers` → `Route by Customer Action`
    - `Route by Customer Action` create output → `Post Create to Linear`
    - `Route by Customer Action` update output → `Post Update to Linear`

20. **Add a Slack node**
    - Node type: `Slack`
    - Name: `Post Slack Notification`
    - Configure Slack credentials
    - Message type: `Block`
    - Channel selection mode: by name
    - Channel: `#your-slack-channel`
    - Text: `=Linear Customer Sync — Run Summary`
    - Blocks expression:

      ```json
      {
        "blocks": [
          {
            "type": "header",
            "text": { "type": "plain_text", "text": "🔄 Linear Customer Sync", "emoji": true }
          },
          {
            "type": "section",
            "fields": [
              { "type": "mrkdwn", "text": "*{{ $json.action.toTitleCase() }}*: `{{ $json.snowflakeData.name }}` (`${{ $json.snowflakeData.arr }}`, `{{ $json.snowflakeData.seats }} seats`)" }
            ]
          }
        ]
      }
      ```

21. **Configure Slack error handling**
    - Set `On Error` to continue regular output
    - Disable retry on fail if you want behavior identical to the original workflow

22. **Connect Slack notifications**
    - `Match and Update Customers` → `Post Slack Notification`

23. **Optional: add sticky notes**
    - Add notes for:
      - overall workflow description
      - trigger and initial fetch
      - pagination
      - data processing
      - matching and routing
      - Linear management
      - Slack notification

24. **Configure workflow settings if needed**
    - Save successful executions: enabled
    - Save error executions: all
    - Save execution progress: enabled
    - Execution order: `v1`
    - Optional error workflow: set a separate workflow ID if your environment uses one

25. **Test the workflow manually**
    - Run it once
    - Validate:
      - Snowflake returns company data
      - Linear pagination completes
      - action items are produced
      - create/update requests succeed
      - Slack messages appear

26. **Activate the workflow**
    - Once credentials and outputs are validated, activate the workflow for scheduled execution

### Credential Setup Summary
- **Snowflake credentials**
  - Required for `Retrieve HubSpot Data from Snowflake`
- **Linear API credentials**
  - Required for `Fetch Customer Data`
  - Required for `Post Create to Linear`
  - Required for `Post Update to Linear`
- **Slack credentials**
  - Required for `Post Slack Notification`

### Important Rebuild Constraints
- Keep node names unchanged if reusing the provided code exactly:
  - `Limit HubSpot Records`
  - `Set Initial Pagination`
  - `Set Next Page Parameters`
  - `Format Output Data`
- The matching code relies on direct node-name lookups
- The workflow depends on the disabled `Limit HubSpot Records` node as a pass-through source reference
- Slack notifications are sent from action generation, not from create/update success responses

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Overall workflow purpose: sync HubSpot-derived Snowflake customer data into Linear daily, then send Slack updates. | Workflow-level note |
| Setup checklist included in the workflow: configure Snowflake, Linear API, Slack credentials, and schedule frequency. | Workflow-level note |
| Slack notifications reflect generated actions, not confirmed successful writes to Linear. | Operational consideration |
| Matching logic uses external ID first and domain as fallback. This improves resilience when company IDs change upstream. | Design note |
| Only revenue and seat changes trigger updates. Customer names, domains, and external IDs are not updated for already matched Linear customers. | Functional limitation |
| The disabled Limit node is still referenced by name in Code nodes, so renaming or removing it requires code updates. | Implementation constraint |