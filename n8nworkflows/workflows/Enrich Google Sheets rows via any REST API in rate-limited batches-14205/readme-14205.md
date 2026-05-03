Enrich Google Sheets rows via any REST API in rate-limited batches

https://n8nworkflows.xyz/workflows/enrich-google-sheets-rows-via-any-rest-api-in-rate-limited-batches-14205


# Enrich Google Sheets rows via any REST API in rate-limited batches

# 1. Workflow Overview

This workflow enriches rows from a Google Sheet by calling an external REST API for each row, processing the rows in controlled batches to avoid API rate-limit issues, and then writing successful enrichment results back into the sheet while capturing failed API calls separately.

Typical use cases include:
- lead enrichment
- product or catalog enrichment
- URL or domain analysis
- any row-by-row spreadsheet augmentation using a REST endpoint

The workflow has a single manual entry point and is organized into a loop that reads rows, processes them batch by batch, pauses between batches, branches on API success or failure, and then continues until all rows are processed.

## 1.1 Trigger and Sheet Input

The workflow starts manually, then reads rows from a configured Google Sheets tab.

## 1.2 Batch Control Loop

The rows are fed into a `Split In Batches` node, which controls how many rows are processed per cycle and enables looping until all rows are exhausted.

## 1.3 API Enrichment Call

Each row in the current batch is sent to an external API using an HTTP request authenticated with Header Auth credentials.

## 1.4 Rate-Limit Handling and Response Evaluation

After the API call, the workflow waits for a fixed pause period, then checks whether the HTTP response returned status code `200`.

## 1.5 Success Writeback and Error Capture

Successful responses update the matching row in Google Sheets with enrichment data and status. Failed responses are transformed into a compact error object containing the row identifier and failure details.

## 1.6 Result Merge and Loop Continuation

Success and error branches are merged and sent back into the batching node so the next batch can be processed.

---

# 2. Block-by-Block Analysis

## Block 1: Trigger and Source Data Retrieval

### Overview:
This block launches the workflow manually and loads source records from Google Sheets. It is the entry point for the entire enrichment loop.

### Nodes Involved:
- `Run Enrichment`
- `Read Rows`

### Node Details

#### 1. Run Enrichment
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual execution trigger.
- **Configuration choices:** No additional parameters are configured. It simply starts the workflow when run manually in the editor or through manual execution.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Read Rows`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No functional failure during configuration, but it is not suitable for automated scheduling by itself.
- **Sub-workflow reference:** None.

#### 2. Read Rows
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a Google Sheets document.
- **Configuration choices:**
  - Document ID is set to `YOUR_GOOGLE_SHEET_ID`
  - Sheet name is `Sheet1`
  - No advanced options are configured
  - Since no explicit operation is set in the JSON, this node is functioning as a read operation using the default Google Sheets read/list behavior for this node version
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**
  - Input: `Run Enrichment`
  - Output: `Batch Rows`
- **Version-specific requirements:** Type version `4.5`; requires a valid Google Sheets OAuth2 credential.
- **Edge cases or potential failure types:**
  - Invalid or inaccessible spreadsheet ID
  - Wrong sheet/tab name
  - OAuth2 credential authorization failure
  - Empty sheet producing no items
  - Missing expected columns such as `id` or `row_number`, which are used later
- **Sub-workflow reference:** None.

---

## Block 2: Batch Control and Iteration

### Overview:
This block chunks the input rows into small groups and drives the iterative loop. It is the key mechanism used to respect API limits and process all rows gradually.

### Nodes Involved:
- `Batch Rows`
- `Merge Results`

### Node Details

#### 3. Batch Rows
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; emits items in batches and supports loop continuation.
- **Configuration choices:**
  - Batch size is set to `5`
  - No extra options are set
  - The node’s second output is used to emit the current batch into the processing chain
  - The first input is used to continue the iteration after downstream processing finishes
- **Key expressions or variables used:** None in configuration, but downstream nodes reference this node via expressions such as:
  - `$('Batch Rows').item.json.row_number`
  - `$('Batch Rows').item.json.id`
- **Input and output connections:**
  - Input from `Read Rows`
  - Loopback input from `Merge Results`
  - Output to `Call API` from output index 1
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If the sheet returns no rows, no batch processing occurs
  - If downstream references assume columns that do not exist in each item, later expression failures may occur
  - Misunderstanding of split-in-batches loop semantics can break the loop if reconnection is changed
- **Sub-workflow reference:** None.

#### 4. Merge Results
- **Type and technical role:** `n8n-nodes-base.merge`; combines results from success and error paths so the loop can continue.
- **Configuration choices:**
  - No explicit mode is configured in the JSON, so default merge behavior applies for this node version
  - Input 0 receives success-path data
  - Input 1 receives error-path data
  - Output is routed back to `Batch Rows` to trigger processing of the next batch
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Inputs: `Update Row`, `Capture Error`
  - Output: `Batch Rows`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If one branch emits no item when the other does, merge behavior may affect timing depending on default mode
  - If edited incorrectly, loop continuation may stop
- **Sub-workflow reference:** None.

---

## Block 3: External API Enrichment

### Overview:
This block sends each current row to the configured REST API endpoint. The request is authenticated using Header Auth credentials and passes the row’s `id` as a query parameter.

### Nodes Involved:
- `Call API`

### Node Details

#### 5. Call API
- **Type and technical role:** `n8n-nodes-base.httpRequest`; performs the outbound REST API call.
- **Configuration choices:**
  - URL: `https://api.example.com/enrich`
  - Query parameters enabled
  - Sends `id={{ $json.id }}`
  - Authentication uses predefined credential type
  - Credential type is `httpHeaderAuth`
  - Response handling is configured to return the **full HTTP response**, not only the body
- **Key expressions or variables used:**
  - Query parameter `id`: `={{ $json.id }}`
  - This requires each incoming row from Google Sheets to contain a field named `id`
- **Input and output connections:**
  - Input: `Batch Rows`
  - Output: `Rate Limit Pause`
- **Version-specific requirements:** Type version `4.3`; requires an `HTTP Header Auth` credential configured with the expected API header name and value.
- **Edge cases or potential failure types:**
  - Missing `id` field in source row
  - Authentication/header misconfiguration
  - DNS or network failure
  - Timeout or endpoint unavailability
  - Non-200 responses
  - Response body shape not matching later expectations, especially `body.result`
  - Depending on HTTP Request settings and n8n error behavior, certain transport-level failures may stop execution before reaching the `IF` node
- **Sub-workflow reference:** None.

---

## Block 4: Rate Limiting and Response Validation

### Overview:
This block enforces a pause after each API call chain and evaluates whether the API response should be treated as successful. It separates normal enrichment writes from error handling.

### Nodes Involved:
- `Rate Limit Pause`
- `API Success?`

### Node Details

#### 6. Rate Limit Pause
- **Type and technical role:** `n8n-nodes-base.wait`; delays execution between API processing cycles.
- **Configuration choices:**
  - Wait amount is `1`
  - The sticky note indicates this is intended as a one-second pause between batches to respect rate limits
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Call API`
  - Output: `API Success?`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If API rate limits are stricter than expected, one second may be insufficient
  - If throughput needs are higher, this pause may be unnecessarily conservative
  - Long-running workflows can accumulate execution time for large sheets
- **Sub-workflow reference:** None.

#### 7. API Success?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the HTTP status code equals `200`.
- **Configuration choices:**
  - Condition compares `{{$json.statusCode}}` to numeric value `200`
  - Strict type validation is enabled
  - Case sensitivity is enabled, though irrelevant here because the comparison is numeric
- **Key expressions or variables used:**
  - Left value: `={{ $json.statusCode }}`
- **Input and output connections:**
  - Input: `Rate Limit Pause`
  - True output: `Update Row`
  - False output: `Capture Error`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If full response is not returned by the HTTP node, `statusCode` would be unavailable
  - If API returns `201`, `204`, or another valid success code, it will incorrectly route to failure
  - If `statusCode` is missing or not numeric, strict validation may cause unexpected behavior
- **Sub-workflow reference:** None.

---

## Block 5: Success Path — Write Enrichment Back to Google Sheets

### Overview:
This block updates the original spreadsheet row with enrichment output and a success status. It relies on matching the original row by `row_number`.

### Nodes Involved:
- `Update Row`

### Node Details

#### 8. Update Row
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates an existing row in Google Sheets.
- **Configuration choices:**
  - Operation: `update`
  - Document ID: `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Sheet1`
  - Mapping mode: manually defined below
  - Matching column: `row_number`
  - Columns written:
    - `row_number = {{ $('Batch Rows').item.json.row_number }}`
    - `enriched_field = {{ $json.body.result }}`
    - `enrichment_status = success`
- **Key expressions or variables used:**
  - `={{ $('Batch Rows').item.json.row_number }}`
  - `={{ $json.body.result }}`
- **Input and output connections:**
  - Input: `API Success?` true branch
  - Output: `Merge Results`
- **Version-specific requirements:** Type version `4.5`; requires the same Google Sheets OAuth2 credential as the read node.
- **Edge cases or potential failure types:**
  - Missing `row_number` in source sheet
  - If `row_number` is not unique, the wrong row could be updated or update behavior may be ambiguous
  - If the API body does not contain `result`, `enriched_field` may be empty or expression evaluation may fail
  - Spreadsheet write permission issues
  - Column names in the sheet must match the configured write fields
- **Sub-workflow reference:** None.

---

## Block 6: Failure Path — Capture API Errors

### Overview:
This block normalizes failed API responses into a compact error object. It does not write errors back to the sheet; it only prepares the failure data for merging and loop continuation.

### Nodes Involved:
- `Capture Error`

### Node Details

#### 9. Capture Error
- **Type and technical role:** `n8n-nodes-base.set`; creates a simplified error record.
- **Configuration choices:**
  - Assigns three fields:
    - `row_id` from the original batch item’s `id`
    - `error_status` from the API response status code
    - `error_message` as a string describing the failure
- **Key expressions or variables used:**
  - `={{ $('Batch Rows').item.json.id }}`
  - `={{ $json.statusCode }}`
  - `API call failed — status {{ $json.statusCode }}`
- **Input and output connections:**
  - Input: `API Success?` false branch
  - Output: `Merge Results`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If the source row has no `id`, `row_id` will be empty
  - The `error_message` value uses inline placeholder syntax inside a plain string; depending on the Set node field mode, this may remain literal unless configured as an expression-capable value. If not interpreted, the stored string may literally be `API call failed — status {{ $json.statusCode }}`
  - Transport-level HTTP failures may prevent this node from running unless the HTTP node is configured to continue on fail or to return structured error output
- **Sub-workflow reference:** None.

---

## Block 7: Inline Documentation / Sticky Notes

### Overview:
These nodes are non-executable annotations embedded in the canvas. They explain setup, credential requirements, and rate-limit tuning.

### Nodes Involved:
- `Template Overview`
- `Batch Config Instructions`
- `API Credential Instructions`

### Node Details

#### 10. Template Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:** Contains a full workflow description, setup requirements, and implementation notes.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; informational only.
- **Sub-workflow reference:** None.

#### 11. Batch Config Instructions
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:** Explains how to change batch size and wait duration.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; informational only.
- **Sub-workflow reference:** None.

#### 12. API Credential Instructions
- **Type and technical role:** `n8n-nodes-base.stickyNote`; canvas documentation.
- **Configuration choices:** Explains HTTP Header Auth and Google Sheets credential setup.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; informational only.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Enrichment | Manual Trigger | Manual workflow start |  | Read Rows |  |
| Read Rows | Google Sheets | Read source rows from the spreadsheet | Run Enrichment | Batch Rows | ## Enrich Google Sheets via API — Rate-Limited Batches\nThis template reads rows from a Google Sheet, calls any external API to enrich each row, and writes results back, in rate-limited batches to avoid hitting API rate limits.\nWho it's for: Developers, marketers, and operations teams who need to enrich spreadsheet data (leads, products, URLs) via any REST API without triggering rate-limit errors.\nHow it works: Read rows from your input Google Sheet tab; split into batches; call your API for each row in the batch; wait 1 second between batches; check the response; write enriched fields back on success; capture errors on failure.\nRequirements: Google Sheets OAuth2 credential; HTTP Header Auth credential.\nHow to set up: connect Google Sheets OAuth2 to both Sheet nodes; set Sheet ID and Tab name in Read Rows; set API endpoint URL in Call API; connect Header Auth in Call API; update Update Row columns; adjust batch size and wait duration using the sticky notes below. |
| Batch Rows | Split In Batches | Control batch size and workflow looping | Read Rows, Merge Results | Call API | ## Batch Size & Rate Limit Config\nSplitInBatches node (`Batch Rows`): set `Batch Size`; default 5 rows per batch; increase for fast APIs; decrease for strict rate limits.\nWait node (`Rate Limit Pause`): default 1 second between batches; change `Amount` to match your API's rate limit. |
| Call API | HTTP Request | Call external enrichment API | Batch Rows | Rate Limit Pause | ## API Credential Setup\nCall API node (`Call API`): set `URL`; set `Method`; under Authentication select `Header Auth`; create a Header Auth credential with the required header name and API key value.\nGoogle Sheets nodes: both Sheet nodes need the same Google Sheets OAuth2 credential; grant access to the spreadsheet in Google Drive. |
| Rate Limit Pause | Wait | Pause execution to respect rate limits | Call API | API Success? | ## Batch Size & Rate Limit Config\nSplitInBatches node (`Batch Rows`): set `Batch Size`; default 5 rows per batch; increase for fast APIs; decrease for strict rate limits.\nWait node (`Rate Limit Pause`): default 1 second between batches; change `Amount` to match your API's rate limit. |
| API Success? | IF | Branch based on HTTP status code | Rate Limit Pause | Update Row, Capture Error | ## API Credential Setup\nCall API node (`Call API`): set `URL`; set `Method`; under Authentication select `Header Auth`; create a Header Auth credential with the required header name and API key value.\nGoogle Sheets nodes: both Sheet nodes need the same Google Sheets OAuth2 credential; grant access to the spreadsheet in Google Drive. |
| Update Row | Google Sheets | Update the spreadsheet with enrichment output | API Success? | Merge Results | ## API Credential Setup\nCall API node (`Call API`): set `URL`; set `Method`; under Authentication select `Header Auth`; create a Header Auth credential with the required header name and API key value.\nGoogle Sheets nodes: both Sheet nodes need the same Google Sheets OAuth2 credential; grant access to the spreadsheet in Google Drive. |
| Capture Error | Set | Build structured error output for failed API calls | API Success? | Merge Results | ## API Credential Setup\nCall API node (`Call API`): set `URL`; set `Method`; under Authentication select `Header Auth`; create a Header Auth credential with the required header name and API key value.\nGoogle Sheets nodes: both Sheet nodes need the same Google Sheets OAuth2 credential; grant access to the spreadsheet in Google Drive. |
| Merge Results | Merge | Rejoin branches and continue batch loop | Update Row, Capture Error | Batch Rows |  |
| Template Overview | Sticky Note | Canvas documentation |  |  | ## Enrich Google Sheets via API — Rate-Limited Batches\nThis template reads rows from a Google Sheet, calls any external API to enrich each row, and writes results back, in rate-limited batches to avoid hitting API rate limits.\nWho it's for: Developers, marketers, and operations teams who need to enrich spreadsheet data (leads, products, URLs) via any REST API without triggering rate-limit errors.\nHow it works: Read rows from your input Google Sheet tab; split into batches; call your API for each row in the batch; wait 1 second between batches; check the response; write enriched fields back on success; capture errors on failure.\nRequirements: Google Sheets OAuth2 credential; HTTP Header Auth credential.\nHow to set up: connect Google Sheets OAuth2 to both Sheet nodes; set Sheet ID and Tab name in Read Rows; set API endpoint URL in Call API; connect Header Auth in Call API; update Update Row columns; adjust batch size and wait duration using the sticky notes below. |
| Batch Config Instructions | Sticky Note | Canvas documentation for batch and wait tuning |  |  | ## Batch Size & Rate Limit Config\nSplitInBatches node (`Batch Rows`): set `Batch Size`; default 5 rows per batch; increase for fast APIs; decrease for strict rate limits.\nWait node (`Rate Limit Pause`): default 1 second between batches; change `Amount` to match your API's rate limit. |
| API Credential Instructions | Sticky Note | Canvas documentation for credential setup |  |  | ## API Credential Setup\nCall API node (`Call API`): set `URL`; set `Method`; under Authentication select `Header Auth`; create a Header Auth credential with the required header name and API key value.\nGoogle Sheets nodes: both Sheet nodes need the same Google Sheets OAuth2 credential; grant access to the spreadsheet in Google Drive. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name it `Run Enrichment`

3. **Add a Google Sheets node for reading**
   - Node type: `Google Sheets`
   - Name it `Read Rows`
   - Connect `Run Enrichment -> Read Rows`
   - Configure:
     - Credential: `Google Sheets OAuth2`
     - Spreadsheet / Document ID: your Google Sheet ID
     - Sheet / Tab name: for example `Sheet1`
   - Ensure the source sheet includes at minimum:
     - `id` column for API lookup
     - `row_number` column for matching updates later

4. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name it `Batch Rows`
   - Connect `Read Rows -> Batch Rows`
   - Set:
     - `Batch Size = 5`
   - This value can be increased or decreased depending on API limits.

5. **Add an HTTP Request node**
   - Node type: `HTTP Request`
   - Name it `Call API`
   - Connect the second/main batch-processing output of `Batch Rows` to `Call API`
   - Configure:
     - URL: your enrichment endpoint, for example `https://api.example.com/enrich`
     - Method: choose what your API expects, commonly `GET` or `POST`
     - Authentication: `Predefined Credential Type`
     - Credential type: `HTTP Header Auth`
     - Query parameters:
       - `id = {{ $json.id }}`
   - In response settings, enable returning the **full response** so the workflow can access:
     - `statusCode`
     - `body`
   - Create or attach a credential:
     - Type: `HTTP Header Auth`
     - Header name: for example `X-API-Key`
     - Header value: your API key

6. **Add a Wait node**
   - Node type: `Wait`
   - Name it `Rate Limit Pause`
   - Connect `Call API -> Rate Limit Pause`
   - Set:
     - `Amount = 1`
   - Interpret this as one second unless your node UI asks for a unit separately; set the unit to seconds if needed in your n8n version.

7. **Add an IF node**
   - Node type: `IF`
   - Name it `API Success?`
   - Connect `Rate Limit Pause -> API Success?`
   - Configure one condition:
     - Left value: `{{ $json.statusCode }}`
     - Operator: `equals`
     - Right value: `200`
     - Value type: number
   - Keep strict type validation enabled if available.

8. **Add a Google Sheets node for updates**
   - Node type: `Google Sheets`
   - Name it `Update Row`
   - Connect the **true** output of `API Success?` to `Update Row`
   - Configure:
     - Credential: same `Google Sheets OAuth2`
     - Operation: `Update`
     - Document ID: same spreadsheet ID
     - Sheet name: same tab, e.g. `Sheet1`
     - Matching column: `row_number`
     - Map fields manually:
       - `row_number = {{ $('Batch Rows').item.json.row_number }}`
       - `enriched_field = {{ $json.body.result }}`
       - `enrichment_status = success`
   - Make sure your target sheet contains the columns:
     - `row_number`
     - `enriched_field`
     - `enrichment_status`

9. **Add a Set node for failures**
   - Node type: `Set`
   - Name it `Capture Error`
   - Connect the **false** output of `API Success?` to `Capture Error`
   - Add fields:
     - `row_id` as string: `{{ $('Batch Rows').item.json.id }}`
     - `error_status` as number: `{{ $json.statusCode }}`
     - `error_message` as string: preferably configure as an expression, for example `{{ 'API call failed — status ' + $json.statusCode }}`
   - This expression form is safer than plain text with curly braces embedded.

10. **Add a Merge node**
    - Node type: `Merge`
    - Name it `Merge Results`
    - Connect:
      - `Update Row -> Merge Results` input 1
      - `Capture Error -> Merge Results` input 2
    - Leave default mode unless you need custom merge behavior.

11. **Close the batch loop**
    - Connect `Merge Results -> Batch Rows`
    - This loopback is what causes the next batch to be emitted after the current branch finishes.

12. **Optional: add sticky notes for maintainability**
    - Add notes describing:
      - workflow purpose
      - batch-size tuning
      - credential setup
    - This workflow includes three such notes:
      - general overview
      - batch/rate-limit configuration
      - API credential configuration

13. **Create credentials**
    - **Google Sheets OAuth2**
      - Authorize the Google account
      - Confirm the account can access the target spreadsheet in Google Drive
    - **HTTP Header Auth**
      - Set the exact header key expected by the API, such as `X-API-Key` or `Authorization`
      - Set the value to the API secret or token

14. **Prepare the source Google Sheet**
    - At minimum include:
      - `id`
      - `row_number`
    - For writeback also include:
      - `enriched_field`
      - `enrichment_status`
    - Ensure `row_number` values uniquely identify rows if you use it as the matching key.

15. **Run a test execution**
    - Start with a few sample rows
    - Check:
      - API request formatting
      - response shape, especially `body.result`
      - whether updates hit the correct rows
      - whether failures route correctly to `Capture Error`

16. **Adjust for your API**
    - If the API expects POST body instead of query params:
      - move `id` into the request body
    - If success codes can be other than `200`:
      - expand the IF logic accordingly
    - If the API returns a different response shape:
      - update `{{ $json.body.result }}` to the correct path
    - If rate limits are stricter:
      - lower batch size and/or increase wait duration

17. **Recommended hardening before production**
    - Enable HTTP node options to continue on fail if you want all failures to route through the error branch instead of stopping the workflow
    - Consider writing failures back to a dedicated sheet or database
    - Consider logging raw API responses for debugging
    - Consider replacing `row_number` with a more stable unique key if row order may change

### Sub-workflow setup
This workflow does **not** use any sub-workflow or workflow-invocation node. No separate child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed as a reusable template for enriching spreadsheet rows through any REST API while reducing rate-limit risk via batching and pauses. | General design note |
| The Google Sheets read and write nodes are expected to use the same OAuth2 credential and target the same spreadsheet/tab unless intentionally changed. | Credential consistency |
| The HTTP Request node is configured to return the full response so the IF node can inspect `statusCode` and the success path can read `body.result`. | Important implementation detail |
| The failure path currently captures error data in the execution stream only; it does not persist failures back to Google Sheets or another datastore unless you extend the workflow. | Operational note |
| The batch loop depends on the `Merge Results -> Batch Rows` connection. Removing or miswiring this connection will stop iterative processing. | Loop control note |