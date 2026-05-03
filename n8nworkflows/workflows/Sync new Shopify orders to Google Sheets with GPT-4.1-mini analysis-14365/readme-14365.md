Sync new Shopify orders to Google Sheets with GPT-4.1-mini analysis

https://n8nworkflows.xyz/workflows/sync-new-shopify-orders-to-google-sheets-with-gpt-4-1-mini-analysis-14365


# Sync new Shopify orders to Google Sheets with GPT-4.1-mini analysis

## 1. Workflow Overview

This workflow automatically polls Shopify for newly created orders, enriches each new order with an AI-generated classification using GPT-4.1-mini, and appends the resulting structured data into a Google Sheet.

Its main use cases are:
- maintaining a lightweight order-tracking sheet for operations or support teams
- enriching raw e-commerce orders with AI-derived metadata such as category, priority, and internal notes
- avoiding duplicate processing by storing the last successful sync timestamp in workflow static data

### 1.1 Scheduled Triggering
The workflow starts automatically every few minutes using a Schedule Trigger.

### 1.2 Sync State Management
A Code node reads the workflow’s global static data to retrieve the timestamp of the last successful sync. If none exists, it initializes the value to the start of the current day.

### 1.3 Shopify Order Retrieval and New-Order Filtering
The workflow requests all Shopify orders created since the stored sync time, marks each returned order as new or old by comparing `created_at` against the saved timestamp, and allows only truly new orders to continue.

### 1.4 AI-Based Order Analysis
For each new order, the workflow sends selected order details to an AI Agent powered by the OpenAI Chat Model `gpt-4.1-mini`. The AI is instructed to return raw JSON containing:
- `category`
- `priority`
- `internal_note`

### 1.5 Data Shaping and Storage
The AI response is parsed, combined with selected Shopify order fields, and formatted into a clean record for insertion into a Google Sheet.

### 1.6 Sync Timestamp Update
After the Google Sheets append succeeds, the workflow updates the global sync timestamp so later runs only fetch newer orders.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Triggering

**Overview:**  
This block is the workflow entry point. It launches the workflow on a recurring interval without manual intervention.

**Nodes Involved:**
- Run Workflow Every Few Minutes

### Node Details

#### Run Workflow Every Few Minutes
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry-point trigger node that starts the workflow on a time interval.
- **Configuration choices:**  
  The trigger is configured with a rule based on an interval in minutes. No custom minute value is explicitly shown in the JSON, so it uses the schedule configuration present in the node UI at runtime.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get Last Successful Sync Time`
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - Workflow inactive: it will not run automatically until activated.
  - Misconfigured schedule frequency may cause overly frequent polling or insufficient freshness.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Sync State Management

**Overview:**  
This block manages persistence of the last successful sync timestamp using workflow global static data. It ensures the workflow can resume from the previous checkpoint.

**Nodes Involved:**
- Get Last Successful Sync Time

### Node Details

#### Get Last Successful Sync Time
- **Type and technical role:** `n8n-nodes-base.code`  
  Reads and initializes workflow-level static state.
- **Configuration choices:**  
  The node accesses `$getWorkflowStaticData('global')`. If `data.lastSync` does not exist, it sets it to the start of the current day in ISO format.
- **Key expressions or variables used:**  
  - `$getWorkflowStaticData('global')`
  - `data.lastSync`
  - `today.toISOString()`
- **Input and output connections:**  
  - Input: `Run Workflow Every Few Minutes`
  - Output: `Fetch Orders from Shopify`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Static data persistence depends on n8n execution/storage behavior.
  - If the instance is reset or migrated, `lastSync` may be lost.
  - Initializing to start of day means the first run may fetch all orders from today, not from an earlier historical range.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Shopify Order Retrieval and New-Order Filtering

**Overview:**  
This block fetches candidate orders from Shopify using the stored sync time and then performs an additional order-by-order comparison to ensure only newly created orders proceed.

**Nodes Involved:**
- Fetch Orders from Shopify
- Mark Orders as New or Old
- Is This a New Order?

### Node Details

#### Fetch Orders from Shopify
- **Type and technical role:** `n8n-nodes-base.shopify`  
  Calls Shopify to retrieve orders.
- **Configuration choices:**  
  - Operation: `getAll`
  - Authentication: access token
  - Order status: `any`
  - `createdAtMin` is dynamically set from `{{ $json.lastSync }}`
- **Key expressions or variables used:**  
  - `={{ $json.lastSync }}`
- **Input and output connections:**  
  - Input: `Get Last Successful Sync Time`
  - Output: `Mark Orders as New or Old`
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Invalid Shopify credentials or expired token
  - API rate limiting
  - Empty result set when no orders match
  - Timestamp formatting mismatch if `lastSync` is malformed
- **Sub-workflow reference:**  
  None.

#### Mark Orders as New or Old
- **Type and technical role:** `n8n-nodes-base.code`  
  Compares each order’s `created_at` timestamp against the saved sync timestamp and adds a boolean flag.
- **Configuration choices:**  
  The node reads global static data again, parses `data.lastSync`, parses `$json.created_at`, and returns the full order object with an extra property:
  - `isNewOrder: createdAt > lastSync`
- **Key expressions or variables used:**  
  - `$getWorkflowStaticData('global')`
  - `$json.created_at`
  - fallback date: `'2025-01-01T00:00:00Z'`
- **Input and output connections:**  
  - Input: `Fetch Orders from Shopify`
  - Output: `Is This a New Order?`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If `created_at` is missing or invalid, date parsing may result in incorrect comparisons.
  - Equal timestamps are treated as not new because the test uses `>` not `>=`.
  - The fallback date is only used if static data is absent here, which could create inconsistent behavior versus the initialization node.
- **Sub-workflow reference:**  
  None.

#### Is This a New Order?
- **Type and technical role:** `n8n-nodes-base.if`  
  Filters items by the computed `isNewOrder` boolean.
- **Configuration choices:**  
  The condition checks whether `{{ $json.isNewOrder }}` is boolean `true`.
- **Key expressions or variables used:**  
  - `={{ $json.isNewOrder }}`
- **Input and output connections:**  
  - Input: `Mark Orders as New or Old`
  - True output: `AI Order Analysis`
  - False output: none connected
- **Version-specific requirements:**  
  `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Strict type validation is enabled, so non-boolean values may fail condition evaluation as intended.
  - Orders judged false simply stop, which is expected behavior.
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI-Based Order Analysis

**Overview:**  
This block enriches each new Shopify order by prompting an AI model to classify the order and generate an internal note.

**Nodes Involved:**
- OpenAI Chat Model
- AI Order Analysis

### Node Details

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model backend used by the AI Agent node.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - Uses OpenAI account credentials
  - No extra model options are defined
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No main input/output
  - AI language model output connected to `AI Order Analysis`
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - Invalid OpenAI API key
  - Quota exhaustion
  - Temporary model/API availability issues
- **Sub-workflow reference:**  
  None.

#### AI Order Analysis
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Executes the AI prompt using the connected OpenAI Chat Model.
- **Configuration choices:**  
  The node defines a prompt using Shopify fields:
  - product name from `fulfillments[0].line_items[0].name`
  - quantity from `fulfillments[0].line_items[0].quantity`
  - total price from `total_price`
  - currency from `currency`
  - country from `shipping_country || 'Unknown'`
  
  It instructs the model to return JSON with:
  - `category`
  - `priority`
  - `internal_note`

  The system message enforces:
  - return only raw JSON
  - no markdown
  - no code fences
- **Key expressions or variables used:**  
  - `{{ $json.fulfillments[0].line_items[0].name }}`
  - `{{ $json.fulfillments[0].line_items[0].quantity }}`
  - `{{ $json.total_price }}`
  - `{{ $json.currency }}`
  - `{{ $json.shipping_country || 'Unknown' }}`
- **Input and output connections:**  
  - Main input: `Is This a New Order?`
  - AI model input: `OpenAI Chat Model`
  - Main output: `Convert AI Response to Data`
- **Version-specific requirements:**  
  `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - `fulfillments[0]` may not exist for unfulfilled orders; this is a major risk.
  - `line_items[0]` may not exist in the assumed path.
  - `shipping_country` is referenced, but Shopify typically provides country via nested shipping address fields; this may often resolve to `Unknown`.
  - The model may still return malformed JSON despite the instruction.
- **Sub-workflow reference:**  
  None.

---

## 2.5 AI Response Parsing and Final Data Preparation

**Overview:**  
This block parses the AI output and merges it with the original Shopify order fields into a final flat schema suitable for spreadsheet storage.

**Nodes Involved:**
- Convert AI Response to Data
- Prepare Final Order Data

### Node Details

#### Convert AI Response to Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the AI agent’s JSON string output into normal structured fields.
- **Configuration choices:**  
  The node assumes the AI result is stored in `$json.output`, then runs `JSON.parse($json.output)`, returning the parsed object as the new item.
- **Key expressions or variables used:**  
  - `$json.output`
  - `JSON.parse(...)`
- **Input and output connections:**  
  - Input: `AI Order Analysis`
  - Output: `Prepare Final Order Data`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Invalid JSON from the AI causes `JSON.parse` to throw and stop execution.
  - If the output property name changes in future LangChain node versions, parsing will fail.
- **Sub-workflow reference:**  
  None.

#### Prepare Final Order Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds the final structured record by combining AI fields with values taken from the order item referenced through the IF node.
- **Configuration choices:**  
  The node assigns these fields:
  - `id`
  - `order_number`
  - `order_created_at`
  - `subtotal_price`
  - `total_price`
  - `currency`
  - `product_name`
  - `quantity`
  - `synced_at`
  - `shipping_country`
  - `category`
  - `Priority`
  - `note`

  It uses expressions referencing `$('Is This a New Order?').item.json...` for the Shopify order and `$json...` for AI-derived fields.
- **Key expressions or variables used:**  
  Examples:
  - `={{ $('Is This a New Order?').item.json.id }}`
  - `={{ $('Is This a New Order?').item.json.fulfillments[0].line_items[0].name }}`
  - `={{ $('Is This a New Order?').item.json.customer.currency }}`
  - `={{$now}}`
  - `={{ $json.category }}`
  - `={{ $json.priority }}`
  - `={{ $json.internal_note }}`
- **Input and output connections:**  
  - Input: `Convert AI Response to Data`
  - Output: `Save Order to Google Sheet`
- **Version-specific requirements:**  
  `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - The reference `$('Is This a New Order?').item.json...` relies on item linking remaining intact.
  - `customer.currency` may be missing; order-level `currency` may be safer.
  - `fulfillments[0].line_items[0]` may be absent.
  - The field name appears as `=synced_at` in the assignment name definition, but downstream mapping expects `synced_at`; this should be verified in the UI because it may be a naming inconsistency.
  - `Priority` is capitalized, but the Google Sheets mapping does not include this field, so it will not be stored.
  - `category` and `note` are also prepared but not mapped to the Google Sheet columns in the current node configuration.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Google Sheets Storage and Sync Timestamp Update

**Overview:**  
This block appends the final order record to Google Sheets and then updates the saved sync time so future runs only process newer orders.

**Nodes Involved:**
- Save Order to Google Sheet
- Update Sync Time

### Node Details

#### Save Order to Google Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends one row per processed order into a target worksheet.
- **Configuration choices:**  
  - Operation: `append`
  - Document: `New Order Track`
  - Sheet: `Shopify_order_data`
  - Mapping mode: define below
  - Explicitly mapped columns:
    - `Id`
    - `Order Number`
    - `order_created`
    - `subtotal_price`
    - `total_price`
    - `currency`
    - `product_name`
    - `quantity`
    - `shipping_country`
    - `Synced_at`
- **Key expressions or variables used:**  
  Examples:
  - `={{ $json.id }}`
  - `={{ $json.order_number }}`
  - `={{ $json.order_created_at }}`
  - `={{ $json.synced_at }}`
- **Input and output connections:**  
  - Input: `Prepare Final Order Data`
  - Output: `Update Sync Time`
- **Version-specific requirements:**  
  `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - Invalid Google Sheets OAuth2 credentials
  - Missing spreadsheet or renamed tab
  - Column mismatch if sheet headers change
  - AI-generated fields are not mapped, so category/priority/note are silently discarded
- **Sub-workflow reference:**  
  None.

#### Update Sync Time
- **Type and technical role:** `n8n-nodes-base.code`  
  Persists the current timestamp as the latest successful sync checkpoint.
- **Configuration choices:**  
  The node writes:
  - `data.lastSync = new Date().toISOString();`
  Then returns existing `items`.
- **Key expressions or variables used:**  
  - `$getWorkflowStaticData('global')`
  - `data.lastSync`
- **Input and output connections:**  
  - Input: `Save Order to Google Sheet`
  - Output: none
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Because this runs after each appended order, the timestamp may update multiple times in one execution.
  - If multiple orders are fetched and one later item fails, some earlier orders may already have advanced `lastSync`.
  - Since it stores the execution time rather than the most recent Shopify order timestamp, late-arriving or backdated orders may be skipped in future runs.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Documentation Sticky Notes

**Overview:**  
These nodes are visual annotations only. They do not affect execution, but they document the workflow’s intent and setup.

**Nodes Involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note for the schedule trigger area.
- **Configuration choices:**  
  Describes the automatic recurring trigger behavior.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note for sync state handling.
- **Configuration choices:**  
  Explains that the workflow remembers the last successful sync time.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note for the new-order detection logic.
- **Configuration choices:**  
  Summarizes how orders are fetched, compared, and deduplicated.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note for processing and storage.
- **Configuration choices:**  
  Explains AI review, cleanup, storage, and sync-time update.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General explanatory note with setup guidance.
- **Configuration choices:**  
  Contains workflow description, behavior summary, and setup steps.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Workflow Every Few Minutes | Schedule Trigger | Starts the workflow on a recurring interval |  | Get Last Successful Sync Time | This node starts the workflow automatically every few minutes.  \nIt checks Shopify regularly for new orders without manual effort. |
| Get Last Successful Sync Time | Code | Reads or initializes the last successful sync timestamp | Run Workflow Every Few Minutes | Fetch Orders from Shopify | This node remembers the last time orders were successfully synced.  \nIt helps avoid fetching the same order again and again. |
| Fetch Orders from Shopify | Shopify | Retrieves Shopify orders created since the stored sync time | Get Last Successful Sync Time | Mark Orders as New or Old | ## New Order Detection Process  \nThis section detects only new Shopify orders.  \nOrders are fetched based on the last successful sync time.  \nEach order’s creation time is compared with the stored sync value.  \nOnly newly created orders are allowed to continue in the workflow.  \nPreviously processed orders are ignored to avoid duplicates. |
| Mark Orders as New or Old | Code | Flags each order as new or old | Fetch Orders from Shopify | Is This a New Order? | ## New Order Detection Process  \nThis section detects only new Shopify orders.  \nOrders are fetched based on the last successful sync time.  \nEach order’s creation time is compared with the stored sync value.  \nOnly newly created orders are allowed to continue in the workflow.  \nPreviously processed orders are ignored to avoid duplicates. |
| Is This a New Order? | If | Filters so only new orders continue | Mark Orders as New or Old | AI Order Analysis | ## New Order Detection Process  \nThis section detects only new Shopify orders.  \nOrders are fetched based on the last successful sync time.  \nEach order’s creation time is compared with the stored sync value.  \nOnly newly created orders are allowed to continue in the workflow.  \nPreviously processed orders are ignored to avoid duplicates. |
| OpenAI Chat Model | OpenAI Chat Model | Provides GPT-4.1-mini to the AI agent |  | AI Order Analysis | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| AI Order Analysis | LangChain Agent | Sends order details to AI for classification and note generation | Is This a New Order? + OpenAI Chat Model | Convert AI Response to Data | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| Convert AI Response to Data | Code | Parses AI JSON output into structured fields | AI Order Analysis | Prepare Final Order Data | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| Prepare Final Order Data | Set | Merges AI fields with Shopify fields into final sheet row structure | Convert AI Response to Data | Save Order to Google Sheet | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| Save Order to Google Sheet | Google Sheets | Appends the final order record to the spreadsheet | Prepare Final Order Data | Update Sync Time | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| Update Sync Time | Code | Saves the current execution time as the latest sync checkpoint | Save Order to Google Sheet |  | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| Sticky Note | Sticky Note | Visual documentation for trigger behavior |  |  | This node starts the workflow automatically every few minutes.  \nIt checks Shopify regularly for new orders without manual effort. |
| Sticky Note1 | Sticky Note | Visual documentation for sync memory behavior |  |  | This node remembers the last time orders were successfully synced.  \nIt helps avoid fetching the same order again and again. |
| Sticky Note2 | Sticky Note | Visual documentation for new-order detection |  |  | ## New Order Detection Process  \nThis section detects only new Shopify orders.  \nOrders are fetched based on the last successful sync time.  \nEach order’s creation time is compared with the stored sync value.  \nOnly newly created orders are allowed to continue in the workflow.  \nPreviously processed orders are ignored to avoid duplicates. |
| Sticky Note3 | Sticky Note | Visual documentation for processing and storage |  |  | ## Order Processing & Storage  \nOnce a new order is detected, this part of the workflow takes over.  \nThe AI reviews the order to understand its importance and category.  \nThe order data is then cleaned, organized, and prepared properly.  \nFinally, the order is saved to Google Sheets and the sync time is updated.  \nThis ensures every order is processed only once and stored safely. |
| Sticky Note4 | Sticky Note | General workflow explanation and setup guidance |  |  | ## How it works  \nThis workflow automatically checks Shopify for new orders at regular intervals.  \nIt remembers the last time it successfully ran and only looks for orders created after that time, so the same order is never processed twice.  \nWhen a new order is found, the workflow extracts important order details such as product name, price, quantity, and shipping country.  \nAn AI step then reviews the order to determine its category, priority level, and adds internal notes for quick understanding.  \nAfter analysis, all order data is cleaned and organized into a final format.  \nThe completed order is saved to Google Sheets as a new row.  \nOnce everything is saved successfully, the workflow updates the last sync time, ensuring future runs only fetch newer orders.  \nIf no new orders are found, the workflow safely stops without errors.  \n## Setup steps  \n**1.** Connect your Shopify credentials to the Shopify node.  \n**2.** Set the Schedule Trigger to decide how often orders should be checked.  \n**3.** Connect your Google Sheets account and select the target sheet.  \n**4.** Review the AI prompt if you want to customize order analysis.  \n**5.** Run the workflow once manually to initialize the first sync time.  \nAfter setup, the workflow runs automatically with no manual effort. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `(Retail) Auto-Sync New Orders to Google Sheets`.
   - Keep execution order at the default `v1` if you want behavior aligned with the exported workflow.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Run Workflow Every Few Minutes`
   - Configure an interval rule using minutes.
   - Choose how often Shopify should be checked.
   - This is the main workflow entry point.

3. **Add a Code node to manage the sync checkpoint**
   - Node type: **Code**
   - Name: `Get Last Successful Sync Time`
   - Paste logic that:
     - reads `$getWorkflowStaticData('global')`
     - checks whether `lastSync` exists
     - if missing, sets it to the start of the current day
     - returns `{ lastSync }`
   - Use code equivalent to:
     - get global static data
     - initialize `lastSync` if absent
     - return one item containing `lastSync`

4. **Connect the trigger to the sync checkpoint node**
   - `Run Workflow Every Few Minutes` → `Get Last Successful Sync Time`

5. **Add a Shopify node**
   - Node type: **Shopify**
   - Name: `Fetch Orders from Shopify`
   - Authentication: **Access Token**
   - Operation: **Get All**
   - In options:
     - set order `status` to `any`
     - set `createdAtMin` to the expression `{{ $json.lastSync }}`
   - Connect Shopify credentials.
   - Select or create a Shopify Access Token credential with the correct store access.

6. **Connect the checkpoint node to Shopify**
   - `Get Last Successful Sync Time` → `Fetch Orders from Shopify`

7. **Add a Code node to mark items as new or old**
   - Node type: **Code**
   - Name: `Mark Orders as New or Old`
   - Configure code that:
     - reads `data.lastSync` from global static data
     - parses the current order’s `created_at`
     - returns the full order plus `isNewOrder: createdAt > lastSync`
   - Include the original order fields in the output by spreading `$json`.

8. **Connect Shopify to the comparison node**
   - `Fetch Orders from Shopify` → `Mark Orders as New or Old`

9. **Add an IF node for new-order filtering**
   - Node type: **If**
   - Name: `Is This a New Order?`
   - Add one condition:
     - left value: `{{ $json.isNewOrder }}`
     - operator: boolean is true
   - Keep strict type validation enabled if you want the same behavior.

10. **Connect the comparison node to the IF node**
    - `Mark Orders as New or Old` → `Is This a New Order?`

11. **Add an OpenAI Chat Model node**
    - Node type: **OpenAI Chat Model**
    - Name: `OpenAI Chat Model`
    - Model: `gpt-4.1-mini`
    - Connect OpenAI credentials.
    - Use an OpenAI API credential with access to the selected model.

12. **Add an AI Agent node**
    - Node type: **AI Agent**
    - Name: `AI Order Analysis`
    - Prompt type: **Define**
    - In the main prompt, include order details from Shopify:
      - product name
      - quantity
      - total price
      - currency
      - country
    - Ask the AI to return JSON with:
      - `category`
      - `priority`
      - `internal_note`
    - Add system message:
      - `You are an API. Return ONLY raw JSON. Do not use markdown. Do not use ``` fences.`
    - Important: connect the Chat Model node to this AI Agent via the AI language model port.

13. **Connect the order filter to the AI agent**
    - True output of `Is This a New Order?` → `AI Order Analysis`

14. **Connect the OpenAI Chat Model to the AI Agent**
    - `OpenAI Chat Model` → AI language model input of `AI Order Analysis`

15. **Add a Code node to parse the AI response**
    - Node type: **Code**
    - Name: `Convert AI Response to Data`
    - Configure it to parse the AI node output:
      - read `$json.output`
      - run `JSON.parse($json.output)`
      - return the parsed object
    - This assumes the AI Agent returns the raw text inside an `output` field.

16. **Connect the AI agent to the parser**
    - `AI Order Analysis` → `Convert AI Response to Data`

17. **Add a Set node to prepare the final row**
    - Node type: **Set**
    - Name: `Prepare Final Order Data`
    - Add fields for:
      - `id`
      - `order_number`
      - `order_created_at`
      - `subtotal_price`
      - `total_price`
      - `currency`
      - `product_name`
      - `quantity`
      - `synced_at`
      - `shipping_country`
      - `category`
      - `Priority`
      - `note`
    - Pull Shopify values using expressions pointing back to the original order item from `Is This a New Order?`
    - Pull AI fields from the current item (`$json.category`, `$json.priority`, `$json.internal_note`)
    - Set `synced_at` to `{{$now}}`

18. **Use these key expressions in the Set node**
    - `id`: `{{ $('Is This a New Order?').item.json.id }}`
    - `order_number`: `{{ $('Is This a New Order?').item.json.order_number }}`
    - `order_created_at`: `{{ $('Is This a New Order?').item.json.created_at }}`
    - `subtotal_price`: `{{ $('Is This a New Order?').item.json.current_subtotal_price }}`
    - `total_price`: `{{ $('Is This a New Order?').item.json.current_total_price }}`
    - `currency`: `{{ $('Is This a New Order?').item.json.customer.currency }}`
    - `product_name`: `{{ $('Is This a New Order?').item.json.fulfillments[0].line_items[0].name }}`
    - `quantity`: `{{ $('Is This a New Order?').item.json.fulfillments[0].line_items[0].current_quantity }}`
    - `shipping_country`: `{{ $json.shipping_address?.country || '' }}` in the exported workflow, but note this references the current AI-parsed item, not the original Shopify item. To reproduce behavior exactly, use that expression; to make it more robust, reference the original Shopify order instead.
    - `category`: `{{ $json.category }}`
    - `Priority`: `{{ $json.priority }}`
    - `note`: `{{ $json.internal_note }}`

19. **Connect the parser to the Set node**
    - `Convert AI Response to Data` → `Prepare Final Order Data`

20. **Prepare the target Google Sheet**
    - Create or select a spreadsheet.
    - Add a worksheet named `Shopify_order_data`.
    - Ensure header columns exist:
      - `Id`
      - `Order Number`
      - `order_created`
      - `subtotal_price`
      - `total_price`
      - `currency`
      - `product_name`
      - `quantity`
      - `shipping_country`
      - `Synced_at`
    - If you also want AI fields stored, add headers like:
      - `category`
      - `priority`
      - `note`
      The exported workflow does not map these, so they are currently omitted from storage.

21. **Add a Google Sheets node**
    - Node type: **Google Sheets**
    - Name: `Save Order to Google Sheet`
    - Operation: **Append**
    - Authenticate with Google Sheets OAuth2.
    - Select the target spreadsheet and worksheet.
    - Use “define below” mapping mode.
    - Map fields:
      - `Id` ← `{{ $json.id }}`
      - `Order Number` ← `{{ $json.order_number }}`
      - `order_created` ← `{{ $json.order_created_at }}`
      - `subtotal_price` ← `{{ $json.subtotal_price }}`
      - `total_price` ← `{{ $json.total_price }}`
      - `currency` ← `{{ $json.currency }}`
      - `product_name` ← `{{ $json.product_name }}`
      - `quantity` ← `{{ $json.quantity }}`
      - `shipping_country` ← `{{ $json.shipping_country }}`
      - `Synced_at` ← `{{ $json.synced_at }}`

22. **Connect the Set node to Google Sheets**
    - `Prepare Final Order Data` → `Save Order to Google Sheet`

23. **Add a final Code node to update sync time**
    - Node type: **Code**
    - Name: `Update Sync Time`
    - Configure it to:
      - read `$getWorkflowStaticData('global')`
      - set `data.lastSync = new Date().toISOString()`
      - return `items`

24. **Connect Google Sheets to the sync update node**
    - `Save Order to Google Sheet` → `Update Sync Time`

25. **Optionally add sticky notes**
    - Add visual notes describing:
      - schedule behavior
      - remembered sync time
      - new-order detection
      - AI processing and Google Sheets storage
      - setup instructions

26. **Configure credentials**
    - **Shopify Access Token**
      - Must allow reading orders.
      - Ensure the store and permissions match the target Shopify shop.
    - **OpenAI API**
      - Must allow use of `gpt-4.1-mini`.
    - **Google Sheets OAuth2**
      - Must have permission to edit the target spreadsheet.

27. **Run a manual test**
    - Execute the workflow manually once.
    - This initializes `lastSync` if not already present.
    - Verify that orders are returned from Shopify.
    - Confirm the AI returns valid JSON.
    - Confirm a row is appended to Google Sheets.

28. **Activate the workflow**
    - Once all credentials and tests are working, activate the workflow so the Schedule Trigger starts polling automatically.

### Important implementation notes when rebuilding
1. **Fulfillment path risk**
   - The workflow uses `fulfillments[0].line_items[0]`.
   - Many Shopify orders may not yet have fulfillments.
   - A more robust production version should use top-level `line_items[0]` if available.

2. **Shipping country inconsistency**
   - The AI prompt uses `shipping_country`, but Shopify usually exposes country via `shipping_address.country`.
   - The final Set node also mixes AI-item and Shopify-item references.
   - Consider standardizing around the original Shopify order path.

3. **AI data is not saved in the exported sheet mapping**
   - `category`, `Priority`, and `note` are prepared but not appended to Google Sheets in the current configuration.
   - If you want them persisted, add matching spreadsheet columns and field mappings.

4. **Sync timestamp strategy**
   - The current design stores the current time after each saved order.
   - A safer alternative is to store the maximum processed Shopify `created_at` timestamp after all items succeed.

5. **JSON parsing hard dependency**
   - If the AI returns anything other than valid JSON, the parser node fails.
   - Consider adding a validation or fallback block in a hardened version.

6. **No sub-workflows are used**
   - This workflow has a single scheduled entry point and does not call any sub-workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically checks Shopify for new orders at regular intervals, enriches them with AI analysis, and stores them in Google Sheets. | General workflow behavior |
| It remembers the last successful sync time to avoid processing the same order twice. | Sync strategy |
| If no new orders are found, the workflow stops safely without errors. | Runtime behavior |
| Setup guidance included in the workflow notes: connect Shopify credentials, configure the schedule, connect Google Sheets, review the AI prompt, and run once manually to initialize sync state. | Project setup guidance |