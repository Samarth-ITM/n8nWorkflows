Sync NetSuite inventory items between NetSuite and Salesforce products

https://n8nworkflows.xyz/workflows/sync-netsuite-inventory-items-between-netsuite-and-salesforce-products-13288


# Sync NetSuite inventory items between NetSuite and Salesforce products

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Synchronize **NetSuite Inventory Items** into **Salesforce Products (Product2)** and associated **Pricebook Entries (PricebookEntry)** using an **upsert-by-external-id** strategy.

**Target use cases:**
- Initial bulk export of NetSuite inventory items into Salesforce.
- Ongoing sync (daily schedule) with paging (20 records per iteration).
- Optional “delta” mode to sync only records modified since the last run.

**High-level logical blocks**
1. **1.1 Trigger + Salesforce Pricebook bootstrap**: Runs on schedule and fetches an active Salesforce Pricebook used for PricebookEntry upserts.
2. **1.2 Paging initialization + loop control**: Initializes offset/hasMore in workflow static data, then loops while NetSuite indicates more records.
3. **1.3 NetSuite extraction (paged list → per-record fetch)**: Pulls a page of Inventory Items (IDs), splits them, then fetches full record details.
4. **1.4 Transform to Salesforce payload**: Maps NetSuite fields into Salesforce upsert payload items.
5. **1.5 Salesforce composite upsert (Product2 + PricebookEntry)**: Uses Salesforce Composite API to upsert Product2 and PricebookEntry in a single request per item.
6. **1.6 Loop bookkeeping + completion**: Updates paging offset, re-checks loop condition, and finally updates lastExportDate when finished.
7. **1.7 Documentation / notes (sticky notes)**: Embedded operator instructions, requirements, and links.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Trigger + Salesforce Pricebook bootstrap
**Overview:** Starts the workflow on a schedule and retrieves an active Salesforce Pricebook. The first Pricebook record returned is referenced later when creating/updating PricebookEntry records.

**Nodes involved:**
- **Execute Workflow Daily**
- **Salesforce: Get Pricebook values**

#### Node: Execute Workflow Daily
- **Type / role:** Schedule Trigger (starts workflow automatically)
- **Configuration (interpreted):**
- Runs on an interval schedule (the JSON shows a default/empty interval object; in the UI this must be set to the desired daily schedule).
- **Inputs/Outputs:** Entry node → outputs to **Salesforce: Get Pricebook values**
- **Potential failures / edge cases:**
  - Misconfigured schedule (never fires or fires too frequently).
  - Timezone differences between business expectations and n8n instance timezone.

#### Node: Salesforce: Get Pricebook values
- **Type / role:** HTTP Request (Salesforce REST API SOQL query)
- **Configuration (interpreted):**
  - GET request to Salesforce REST Query endpoint:
    - `.../services/data/v64.0/query?q=SELECT+Id,Name,IsStandard,IsActive+FROM+Pricebook2+WHERE+IsActive=true`
  - Auth: **Salesforce OAuth2** (predefined credential type in n8n)
- **Key data dependencies:**
  - Later nodes reference: `$('Salesforce: Get Pricebook values').item.json.records[0].Id`
- **Outputs:** → **Init Offset**
- **Potential failures / edge cases:**
  - OAuth token expired / invalid scope.
  - No active pricebooks returned → `records[0]` is undefined, causing expression failures downstream.
  - API version mismatch (v64.0) if org does not support it (rare; usually fine).
  - Large orgs: query returns multiple pricebooks; this workflow implicitly uses the **first**.

---

### Block 1.2 — Paging initialization + loop control
**Overview:** Uses workflow static data to manage paging (`offset`) and continuation (`hasMore`). The workflow loops until `hasMore` becomes false, then ends and updates the stored last export date.

**Nodes involved:**
- **Init Offset**
- **Has More Records?**
- **Retrieve Paging Offset and LastExportDate**
- **Update Paging Offset and LastExportDate**
- **Update LastExportDate**
- **Workflow is finished**

#### Node: Init Offset
- **Type / role:** Code node (initializes loop state in static data)
- **Configuration choices:**
  - Writes to `$getWorkflowStaticData('global')`:
    - `offset = 0`
    - `hasMore = true`
- **Output:** Emits `{ offset, hasMore }`
- **Connections:** Receives from **Salesforce: Get Pricebook values** → outputs to **Has More Records?**
- **Potential failures / edge cases:**
  - Static data persists across executions in n8n; this node resets it for the run, which is good.
  - If workflow is executed concurrently (multiple runs overlapping), global static data can be overwritten (race condition). Consider per-execution state instead if concurrency is possible.

#### Node: Has More Records?
- **Type / role:** IF node (loop continuation check)
- **Configuration (interpreted):**
  - Condition: `{{ $json.hasMore }}` is **true**
- **Outputs:**
  - **True** branch → **Retrieve Paging Offset and LastExportDate**
  - **False** branch → **Update LastExportDate**
- **Potential failures / edge cases:**
  - If `hasMore` is missing/not boolean, strict validation may route unexpectedly.
  - If NetSuite list call fails before updating `hasMore`, the loop may not terminate.

#### Node: Retrieve Paging Offset and LastExportDate
- **Type / role:** Code node (reads loop state + determines delta baseline date)
- **Configuration choices:**
  - Reads static data: `workflowStaticData.offset`
  - Determines `lastExportDate`:
    - If previously stored: use it
    - Else: defaults to **yesterday** (`$today.minus({days: 1}).toFormat('MM/dd/yyyy')`)
- **Output:** `{ offset, lastExportDate }`
- **Connections:** → **NS: Inventory Item - Get list of All records**
- **Potential failures / edge cases:**
  - `workflowStaticData.offset` might be `undefined` if initialization was bypassed.
  - Date format is `MM/dd/yyyy`; must match NetSuite query expectations when using delta node.

#### Node: Update Paging Offset and LastExportDate
- **Type / role:** Code node (increments offset and refreshes hasMore from the last NetSuite list call)
- **Configuration choices:**
  - `workflowStaticData.offset = workflowStaticData.offset + 20`
  - Sets `workflowStaticData.hasMore` based on which list node ran:
    - If **NS: Inventory Item - Get list of All records** executed, read its first item `.json.hasMore`
    - Else read **NS: Inventory Item - Get Delta records** `.json.hasMore`
- **Output:** `{ offset, hasMore }`
- **Connections:** Receives from **Salesforce: Add Products** → outputs back to **Has More Records?** (loop)
- **Potential failures / edge cases:**
  - If offset is undefined, `undefined + 20` becomes `NaN`.
  - If neither NetSuite list node executed (or failed), referencing `.first()` may throw.
  - Assumes page size is always 20; if NetSuite node limit changes, increment must match.

#### Node: Update LastExportDate
- **Type / role:** Code node (stores end-of-run timestamp baseline)
- **Configuration choices:**
  - Sets `workflowStaticData.lastExportDate = DateTime.now().toFormat('MM/dd/yyyy')`
- **Connections:** From **Has More Records? (false)** → **Workflow is finished**
- **Potential failures / edge cases:**
  - Depends on Luxon `DateTime` availability in n8n Code node environment (typically available in n8n code node context).
  - If the workflow fails mid-run, lastExportDate is not updated; next delta run may reprocess items.

#### Node: Workflow is finished
- **Type / role:** NoOp (visual terminator)
- **Connections:** End node
- **Potential failures / edge cases:** None (except if used to infer successful completion; it doesn’t enforce it).

---

### Block 1.3 — NetSuite extraction (paged list → per-record fetch)
**Overview:** Pulls NetSuite InventoryItem records in pages of 20. The list call returns an array of items (IDs), which are split into individual items, then each item is fetched with expanded subresources for richer data (e.g., pricing).

**Nodes involved:**
- **NS: Inventory Item - Get list of All records**
- **Split Customers Array** (name is misleading; it splits inventory item list)
- **NS: Inventory Item - Get record**
- *(Optional alternative)* **NS: Inventory Item - Get Delta records** (not connected in current wiring)

#### Node: NS: Inventory Item - Get list of All records
- **Type / role:** NetSuite REST community node (`n8n-nodes-netsuite-rest.netSuiteRest`) list operation
- **Configuration choices:**
  - Resource: `InventoryItem`
  - Additional fields:
    - `limit = 20`
    - `offset = {{ $json.offset }}`
- **Inputs/Outputs:** Receives `{offset,...}` from **Retrieve Paging Offset and LastExportDate** → outputs to **Split Customers Array**
- **Potential failures / edge cases:**
  - NetSuite OAuth2/auth errors; insufficient permissions to InventoryItem.
  - Pagination behavior depends on NetSuite REST API response; expects `hasMore` and an `items` array.
  - Offset may exceed total records; should eventually return `hasMore=false` or empty items.

#### Node: Split Customers Array
- **Type / role:** Split Out (turns array into individual items)
- **Configuration choices:**
  - `fieldToSplitOut: items`
- **Connections:** From **NS: Inventory Item - Get list of All records** → to **NS: Inventory Item - Get record**
- **Potential failures / edge cases:**
  - If `items` is missing or not an array (NetSuite error payload), node fails.
  - If `items` is empty, downstream nodes won’t execute; loop continues based on `hasMore`.

#### Node: NS: Inventory Item - Get record
- **Type / role:** NetSuite REST community node record fetch
- **Configuration choices:**
  - Operation: `get /inventoryItem/{id}`
  - `id = {{ $json.id }}` (from split item)
  - `expandSubResources: true` (to include nested resources like price)
- **Outputs:** → **Prepare Salesforce Payload**
- **Potential failures / edge cases:**
  - Missing `id` in list items.
  - NetSuite record-level access restrictions.
  - Expanded subresources increase payload size; may affect performance/timeouts.

#### Node: NS: Inventory Item - Get Delta records (optional / currently not connected)
- **Type / role:** NetSuite REST list with query filter for lastModifiedDate
- **Configuration choices:**
  - Resource: `InventoryItem`
  - Debug mode enabled
  - `q = lastModifiedDate ON_OR_AFTER "{{ $json.lastExportDate }}"`
  - `limit = 20`
  - `offset = {{ $json.offset }}`
- **Intended wiring:** Replace “Get list of All records” with this node to only process changes since last run.
- **Potential failures / edge cases:**
  - Date format must match what NetSuite expects in `q`.
  - On first scheduled run, sticky note indicates lastExportDate defaults to “today” (in this workflow, the *code* defaults to yesterday unless lastExportDate exists; be aware of this difference between note and implementation).
  - If NetSuite search syntax changes, `q` may return errors.

---

### Block 1.4 — Transform to Salesforce payload
**Overview:** Converts NetSuite InventoryItem records into a simplified object containing Salesforce Product fields and a derived price field from NetSuite pricing.

**Nodes involved:**
- **Prepare Salesforce Payload**

#### Node: Prepare Salesforce Payload
- **Type / role:** Code node (mapping/transformation)
- **Configuration choices:**
  - Iterates through all incoming items (`$input.all()`).
  - Extracts:
    - `Name`: `data.itemId`
    - `External_ID__c`: `data.id` (NetSuite internal ID as Salesforce external ID)
    - `IsActive`: `data.isinactive === "F"`
    - `PricebookEntry_Price__c`: derived from `data.price.items[]` where `priceLevelName === "Quantity"`; uses `.price` or null
- **Outputs:** One item per record with `{Name, External_ID__c, IsActive, PricebookEntry_Price__c}`
- **Connections:** → **Salesforce: Add Products**
- **Potential failures / edge cases:**
  - If `data.price.items` structure differs, `quantityPrice` becomes null (safe), but downstream currently **does not use** this value (see Salesforce node uses `UnitPrice: 100`).
  - If `itemId` missing, Salesforce Name becomes empty (Salesforce may reject).
  - If `isinactive` values differ (e.g., boolean), `IsActive` mapping may be wrong.

---

### Block 1.5 — Salesforce composite upsert (Product2 + PricebookEntry)
**Overview:** For each item, sends a Salesforce Composite API request that upserts Product2 by external ID and then upserts a related PricebookEntry, referencing the product created/updated in the same composite call.

**Nodes involved:**
- **Salesforce: Add Products**

#### Node: Salesforce: Add Products
- **Type / role:** HTTP Request (Salesforce Composite REST API)
- **Configuration choices:**
  - Method: POST
  - URL: `https://<your-domain>.my.salesforce.com/services/data/v64.0/composite`
  - Auth: Salesforce OAuth2 credential
  - Body (Composite request) does:
    1) **PATCH Product2** by external id:
       - URL: `/sobjects/Product2/External_ID__c/{{ $json.External_ID__c }}`
       - Body: sets `Name` and `IsActive: true` (note: ignores mapped IsActive)
    2) **PATCH PricebookEntry** by external id:
       - URL: `/sobjects/PricebookEntry/External_ID__c/{{ $json.External_ID__c }}-{{ pricebookId }}`
       - Body:
         - `Pricebook2Id`: from earlier query `records[0].Id`
         - `Product2Id`: `@{prodUpsert.id}` (reference to composite subrequest result)
         - `UnitPrice`: 100 (hardcoded)
         - `IsActive`: true
  - `allOrNone: true` means either both upserts succeed or both fail.
- **Connections:** Receives from **Prepare Salesforce Payload** → outputs to **Update Paging Offset and LastExportDate**
- **Potential failures / edge cases:**
  - Requires **External_ID__c** fields configured as External ID/unique (or at least usable in upsert URL) on **Product2** and **PricebookEntry**.
  - If Pricebook query returns 0 records, expression for Pricebook2Id fails.
  - Hardcoded `UnitPrice: 100` may be incorrect; if Salesforce requires non-null and business logic differs, update to use mapped `PricebookEntry_Price__c`.
  - Composite reference `@{prodUpsert.id}` depends on Product2 PATCH returning an `id` in composite response (Salesforce does for composite subrequests; still ensure correct API behavior).
  - If `allOrNone=true`, a failure in either upsert rejects both (potentially blocking progress).
  - API limits: composite calls per day, concurrent requests, etc.

---

### Block 1.6 — Loop bookkeeping + completion
**Overview:** After upserting a page of items, the workflow increments the offset and re-checks `hasMore`. When NetSuite indicates no more records, it stores the current date as lastExportDate and ends.

**Nodes involved:**
- **Update Paging Offset and LastExportDate**
- **Has More Records?**
- **Update LastExportDate**
- **Workflow is finished**

*(These nodes were detailed in Block 1.2; functionally they finalize each iteration and terminate the run.)*

---

### Block 1.7 — Embedded notes / operator guidance (Sticky Notes)
**Overview:** Four sticky notes provide requirements, usage guidance, and links relevant to NetSuite query syntax and the community node.

**Nodes involved:**
- **Sticky Note7**
- **Sticky Note**
- **Sticky Note1**
- **Sticky Note2**
- **Sticky Note3**

(Sticky notes are documented in the summary table and “General Notes & Resources”.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note7 | Sticky Note | Operator notes / requirements | — | — | ### This n8n template demonstrates how to export NetSuite Inventory Items and Upsert into Salesforce as Products. |
| Execute Workflow Daily | Schedule Trigger | Workflow entry point (scheduled run) | — | Salesforce: Get Pricebook values |  |
| Salesforce: Get Pricebook values | HTTP Request | Fetch active Salesforce Pricebook2 records | Execute Workflow Daily | Init Offset | ## Salesforce section 1<br>### Salesforce Pricebook values are fetched here<br>### Note<br>* In this workflow only one Salesforce Pricebook value is used |
| Init Offset | Code | Initialize paging loop state in workflow static data | Salesforce: Get Pricebook values | Has More Records? |  |
| Has More Records? | IF | Loop controller based on `hasMore` | Init Offset; Update Paging Offset and LastExportDate | Retrieve Paging Offset and LastExportDate (true); Update LastExportDate (false) |  |
| Retrieve Paging Offset and LastExportDate | Code | Read offset + compute lastExportDate default | Has More Records? (true) | NS: Inventory Item - Get list of All records |  |
| NS: Inventory Item - Get list of All records | NetSuite REST (community) | List InventoryItem records (paged) | Retrieve Paging Offset and LastExportDate | Split Customers Array | ## Netsuite section<br>### Records are pulled from NetSuite here<br>### Fell free to use filter with Q parameter passed in 'NS: Inventory Item - Get list of All records' step<br>Read about using Q parameter in [NetSuite Docs](https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_1545222128.html)<br>### Note<br>* In this workflow only one Netsuite price level value is used |
| Split Customers Array | Split Out | Split `items[]` into individual items | NS: Inventory Item - Get list of All records | NS: Inventory Item - Get record |  |
| NS: Inventory Item - Get record | NetSuite REST (community) | Fetch full InventoryItem record by id | Split Customers Array | Prepare Salesforce Payload | ## Netsuite section<br>### Records are pulled from NetSuite here<br>Read about using Q parameter in [NetSuite Docs](https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_1545222128.html) |
| Prepare Salesforce Payload | Code | Map NetSuite record to Salesforce upsert fields | NS: Inventory Item - Get record | Salesforce: Add Products |  |
| Salesforce: Add Products | HTTP Request | Salesforce Composite upsert for Product2 + PricebookEntry | Prepare Salesforce Payload | Update Paging Offset and LastExportDate | ## Salesforce section 2<br>### Records are inserted/updated into Salesforce here |
| Update Paging Offset and LastExportDate | Code | Increment offset; refresh hasMore for next loop | Salesforce: Add Products | Has More Records? |  |
| Update LastExportDate | Code | Store lastExportDate when finished | Has More Records? (false) | Workflow is finished |  |
| Workflow is finished | NoOp | Visual terminator | Update LastExportDate | — |  |
| NS: Inventory Item - Get Delta records | NetSuite REST (community) | (Optional) List only modified InventoryItems since lastExportDate | — (not connected) | — | ## Run for Delta records<br>### Replace a 'NS: Inventory Item - Get list of All records' node with a node below to run flow only for records changed after Last Workflow Run |

Additional sticky notes present but not directly covering unique nodes in the canvas table above:
- **Sticky Note**: “Salesforce section 2…”
- **Sticky Note1**: “Netsuite section…”
- **Sticky Note2**: “Run for Delta records…”
- **Sticky Note3**: “Salesforce section 1…”

(Their content is included in the rows where relevant, and again in section 5.)

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Salesforce OAuth2 credential** in n8n:
      - Connect to the target Salesforce org.
      - Ensure it can call REST endpoints `/services/data/v64.0/...`.
   2. **NetSuite REST OAuth2 credential** for the community node:
      - Install and enable the community node package **n8n-nodes-netsuite-rest** (self-hosted n8n required).
      - Configure NetSuite OAuth2 per the node’s documentation and your NetSuite account settings.

2) **Add Trigger**
   1. Add **Schedule Trigger** node named **“Execute Workflow Daily”**.
   2. Configure it to run daily (choose time/timezone as needed).

3) **Fetch Salesforce Pricebook**
   1. Add **HTTP Request** node named **“Salesforce: Get Pricebook values”**.
   2. Set **Authentication**: *Predefined Credential Type* → **Salesforce OAuth2** credential.
   3. Method: **GET**
   4. URL:
      - `https://<your-salesforce-domain>.my.salesforce.com/services/data/v64.0/query?q=SELECT+Id,Name,IsStandard,IsActive+FROM+Pricebook2+WHERE+IsActive=true`
   5. Connect: **Execute Workflow Daily → Salesforce: Get Pricebook values**

4) **Initialize paging state**
   1. Add **Code** node named **“Init Offset”** with:
      - Set global static data: `offset = 0`, `hasMore = true`
   2. Connect: **Salesforce: Get Pricebook values → Init Offset**

5) **Loop controller**
   1. Add **IF** node named **“Has More Records?”**
   2. Condition: Boolean “is true” on expression:
      - `{{ $json.hasMore }}`
   3. Connect: **Init Offset → Has More Records?**

6) **Read offset and lastExportDate**
   1. Add **Code** node named **“Retrieve Paging Offset and LastExportDate”**
   2. Logic:
      - Read `workflowStaticData.offset`
      - Compute `lastExportDate` from static data, else default to yesterday formatted `MM/dd/yyyy`
   3. Connect: **Has More Records? (true) → Retrieve Paging Offset and LastExportDate**

7) **NetSuite list (all records mode)**
   1. Add **NetSuite REST** node (community) named **“NS: Inventory Item - Get list of All records”**
   2. Resource: **InventoryItem**
   3. Additional fields:
      - `limit: 20`
      - `offset: {{ $json.offset }}`
   4. Connect: **Retrieve Paging Offset and LastExportDate → NS: Inventory Item - Get list of All records**

8) **Split list into individual items**
   1. Add **Split Out** node named **“Split Customers Array”**
   2. Field to split out: `items`
   3. Connect: **NS: Inventory Item - Get list of All records → Split Customers Array**

9) **Fetch each NetSuite record**
   1. Add **NetSuite REST** node named **“NS: Inventory Item - Get record”**
   2. Operation: `get /inventoryItem/{id}`
   3. ID: `{{ $json.id }}`
   4. Enable/Set: **Expand subresources = true**
   5. Connect: **Split Customers Array → NS: Inventory Item - Get record**

10) **Prepare Salesforce payload**
   1. Add **Code** node named **“Prepare Salesforce Payload”**
   2. Map NetSuite record to:
      - `Name = itemId`
      - `External_ID__c = id`
      - `IsActive = (isinactive === "F")`
      - `PricebookEntry_Price__c` derived from price level “Quantity” if present
   3. Connect: **NS: Inventory Item - Get record → Prepare Salesforce Payload**

11) **Salesforce composite upsert**
   1. Add **HTTP Request** node named **“Salesforce: Add Products”**
   2. Authentication: **Salesforce OAuth2** credential
   3. Method: **POST**
   4. URL:
      - `https://<your-salesforce-domain>.my.salesforce.com/services/data/v64.0/composite`
   5. Body type: **JSON**
   6. JSON body (key points):
      - Composite PATCH Product2 by external id:
        - `/sobjects/Product2/External_ID__c/{{ $json.External_ID__c }}`
      - Composite PATCH PricebookEntry by external id:
        - `/sobjects/PricebookEntry/External_ID__c/{{ $json.External_ID__c }}-{{ $('Salesforce: Get Pricebook values').item.json.records[0].Id }}`
      - Reference product id via `@{prodUpsert.id}`
      - Decide whether to keep `UnitPrice: 100` or replace with `{{ $json.PricebookEntry_Price__c }}`.
   7. Connect: **Prepare Salesforce Payload → Salesforce: Add Products**

12) **Update paging state and loop**
   1. Add **Code** node named **“Update Paging Offset and LastExportDate”**
   2. Implement:
      - `offset += 20`
      - `hasMore = (read from latest NetSuite list node output json.hasMore)`
   3. Connect: **Salesforce: Add Products → Update Paging Offset and LastExportDate**
   4. Connect: **Update Paging Offset and LastExportDate → Has More Records?** (creates the loop)

13) **Completion path**
   1. Add **Code** node named **“Update LastExportDate”**
      - Set `workflowStaticData.lastExportDate = now formatted MM/dd/yyyy`
   2. Add **NoOp** node named **“Workflow is finished”**
   3. Connect: **Has More Records? (false) → Update LastExportDate → Workflow is finished**

14) **(Optional) Delta mode**
   1. Add/Configure **“NS: Inventory Item - Get Delta records”** NetSuite REST node:
      - Resource: InventoryItem
      - Additional fields:
        - `q = lastModifiedDate ON_OR_AFTER "{{ $json.lastExportDate }}"`
        - `limit = 20`, `offset = {{ $json.offset }}`
   2. Replace the connection from **Retrieve Paging Offset and LastExportDate** so it points to **Get Delta records** instead of **Get list of All records**.
   3. Ensure **Update Paging Offset and LastExportDate** reads `hasMore` from whichever node you actually use.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow processes 20 records per iteration until there are no records left. | Sticky Note7 |
| To find unique records, NetSuite Internal Id is used in Salesforce as External Id. | Sticky Note7 |
| Replace the schedule trigger with any other trigger as needed; run manually once for full backfill, then schedule. | Sticky Note7 |
| To process only records updated after last run date, replace “Get list of All records” with “Get Delta records”. | Sticky Note7 / Sticky Note2 |
| Salesforce requirement: External Id field exists (for Product2 and PricebookEntry in this design). | Sticky Note7 |
| NetSuite REST community node details: [https://www.npmjs.com/package/n8n-nodes-netsuite-rest](https://www.npmjs.com/package/n8n-nodes-netsuite-rest) | Sticky Note7 |
| NetSuite Q parameter documentation: [https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_1545222128.html](https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_1545222128.html) | Sticky Note1 |
| Note: only one NetSuite price level value is used; only one Salesforce Pricebook is used. | Sticky Note1 / Sticky Note3 |
| First scheduled run note (operator guidance): workflow will use “today” as last export date. (Implementation currently defaults to yesterday if none is stored.) | Sticky Note7 vs. Retrieve Paging Offset and LastExportDate code |
| Support contact: support@entechsolutions.com | Sticky Note7 |
| Constraint: community NetSuite REST node implies self-hosted n8n only. | Sticky Note7 |