Build a full REST-API with n8n webhooks

https://n8nworkflows.xyz/workflows/build-a-full-rest-api-with-n8n-webhooks-12789


# Build a full REST-API with n8n webhooks

## 1. Workflow Overview

**Title:** Build a full REST-API with n8n webhooks

**Purpose / use cases**
This workflow provides a reusable pattern to implement a REST-style API in n8n using **multiple Webhook nodes** (to support multiple path depths) and **Switch-based routing** by endpoint and HTTP method. It also demonstrates a clean data-handling pattern using “global” node references (`$('NODE').item.json`) to avoid clutter from non–pass-through nodes.

**Logical blocks**
1. **1.1 API Entry & Method Normalization (Noop + Set)**
   - Receives requests on `/api/v1/...` at 1–3 path segment depth and normalizes the HTTP method into a single field.
2. **1.2 Global Request & Config Initialization**
   - Captures request context into `_REQUEST`, initializes `_CFG` (including auto base URL extraction), then clears `$json` to separate “globals” from “payload flow”.
3. **1.3 Endpoint Routing (by path segments)**
   - Routes `lvl1/lvl2/lvl3` to endpoint branches; unknown routes fall back to “Invalid Route” responders.
4. **1.4 Method Routing & Endpoint Implementation (placeholders)**
   - Routes by HTTP method (GET/POST) and calls placeholder sub-workflows (Execute Workflow) before responding.
5. **1.5 Examples: $json vs $('NODE') notation**
   - A separate manual-triggered section illustrating why absolute node references help when using non–pass-through nodes, including how to check whether an optional node executed.

---

## 2. Block-by-Block Analysis

### 2.1 API Entry & Method Normalization

**Overview**
Three Webhook nodes cover 1, 2, or 3 routing segments. Each incoming request is forwarded to one of four “method Set” nodes which annotate a unified `method` field, then converge into a single “API entry point”.

**Nodes involved**
- `v1/seg1`, `v1/seg2`, `v1/seg3`
- `-> GET`, `-> POST`, `-> PUT`, `-> DELETE`
- `API entry point`

#### Node: `v1/seg1`
- **Type / role:** Webhook trigger; path depth 1 router entry.
- **Config (interpreted):**
  - Path: `/api/v1/:lvl1`
  - Methods: GET, POST, PUT, DELETE (multiple methods enabled)
  - Response mode: **Respond via Respond to Webhook node** (`responseNode`)
- **Key data produced:** `params.lvl1`, plus typical `body`, `query`, `headers`, `webhookUrl`.
- **Connections:**
  - Output 0→ `-> GET`
  - Output 1→ `-> POST`
  - Output 2→ `-> PUT`
  - Output 3→ `-> DELETE`
- **Edge cases / failures:**
  - If no Respond node is reached, webhook call will hang or error depending on n8n settings.
  - n8n limitation noted in sticky note: path variables add a hash base and each depth needs its own webhook.
- **Version notes:** Node type version 2.1 (Webhook).

#### Node: `v1/seg2`
- **Type / role:** Webhook trigger; path depth 2 router entry.
- **Config:** Path `/api/v1/:lvl1/:lvl2`, same methods and response mode as above.
- **Connections:** Same method fan-out to `-> GET/POST/PUT/DELETE`.
- **Edge cases:** Same as `v1/seg1`.

#### Node: `v1/seg3`
- **Type / role:** Webhook trigger; path depth 3 router entry.
- **Config:** Path `/api/v1/:lvl1/:lvl2/:lvl3`, same methods and response mode.
- **Connections:** Same method fan-out to `-> GET/POST/PUT/DELETE`.
- **Edge cases:** Same as above.

#### Node: `-> GET`
- **Type / role:** Set node; method normalization.
- **Config:** Adds/overwrites field `method = "GET"`, passes other fields through (`includeOtherFields: true`).
- **Connections:** → `API entry point`
- **Edge cases:** If upstream already sets `method`, this overwrites it intentionally.

#### Node: `-> POST`
- **Type / role:** Set node; method normalization.
- **Config:** `method = "POST"`, pass-through.
- **Connections:** → `API entry point`

#### Node: `-> PUT`
- **Type / role:** Set node; method normalization.
- **Config:** `method = "PUT"`, pass-through.
- **Connections:** → `API entry point`

#### Node: `-> DELETE`
- **Type / role:** Set node; method normalization.
- **Config:** `method = "DELETE"`, pass-through.
- **Connections:** → `API entry point`

#### Node: `API entry point`
- **Type / role:** NoOp; visual/structural entry marker so all methods converge cleanly.
- **Connections:** → `_REQUEST`
- **Edge cases:** None (NoOp).

**Sticky note coverage**
- “API routing levels … To add more … copy the highest-level Webhook … add `/:lvl4` … connect outputs to corresponding mode Set-node.”

---

### 2.2 Global Request & Config Initialization

**Overview**
The workflow stores the incoming request data into `_REQUEST` (minus headers), creates `_CFG` containing defaults and derived `api_base_url`, then clears the main JSON payload to keep subsequent flow “clean” while still being able to access globals via `$('...')`.

**Nodes involved**
- `_REQUEST`
- `_CFG`
- `Clear JSON`

#### Node: `_REQUEST`
- **Type / role:** Set node; creates a global request object accessible everywhere after execution.
- **Config (interpreted):**
  - Includes everything **except** `headers` (explicitly excluded).
  - Passes other fields through.
- **Key result:** `_REQUEST.item.json` contains `body`, `query`, `params` (`lvl1..lvl3` if present), `method` (from earlier Set), `webhookUrl`, etc., but not headers.
- **Connections:** → `_CFG`
- **Edge cases:**
  - If downstream expects headers, they are removed here. (You’d need to keep them or store selectively.)

#### Node: `_CFG`
- **Type / role:** Code node; global configuration and derived values.
- **Config choices:**
  - Runs “once for each item”.
  - Creates a config object with:
    - `debug: true`
    - `defaults.invalid_route.response` + `defaults.invalid_route.status_code` (400)
    - `db.tables.internal_label` mapping placeholder
    - `api_base_url: 'auto'` initially
  - If `api_base_url` is `'auto'`, it derives it from `$(' _REQUEST').item.json.webhookUrl`:
    - Finds `'/api/v'`, extracts version digits, builds `base + /api/v{version}`.
- **Key expressions / variables:**
  - `$(' _REQUEST').item.json.webhookUrl`
  - Throws error if marker not found: `No api base url for "/api/v" in "{full}"`
- **Connections:** → `Clear JSON`
- **Edge cases / failures:**
  - If the webhook URL format changes or doesn’t contain `/api/v`, the node throws and the request will fail (no response unless you add an error handler).
  - If version digits can’t be matched, `version` may be undefined; the template assumes `/api/v1` etc.
- **Version notes:** Code node v2.

#### Node: `Clear JSON`
- **Type / role:** Set node; wipes the current item JSON so subsequent logic doesn’t accidentally rely on flowing request content.
- **Config:** No explicit assignments; with `alwaysOutputData: true` it ensures output exists.
- **Connections:** → `Routing`
- **Edge cases:**
  - Any downstream node using `$json` will now see an empty object unless it uses global references (`$('...')`).

**Sticky note coverage**
- “Streamlining data: initialize global data for _REQUEST, set up _CFG, then clear JSON…”

---

### 2.3 Endpoint Routing (by path segments)

**Overview**
Routing is performed by a hierarchy of Switch nodes checking `params.lvl1`, `params.lvl2`, and `params.lvl3` stored in `_REQUEST`. Unknown routes go to duplicated “Invalid Route” responders to avoid visual clutter.

**Nodes involved**
- `Routing`
- `api root`
- `/foo`
- `/foo/bar`
- `/foo/bar/baz`
- `/foo/qux`
- `Invalid Route`, `Invalid Route1`, `Invalid Route2`, `Invalid Route3`

#### Node: `Routing`
- **Type / role:** NoOp; “router start” marker after initialization.
- **Connections:** → `api root`

#### Node: `api root`
- **Type / role:** Switch; routes based on top-level segment `lvl1`.
- **Config (interpreted):**
  - If `$(' _REQUEST').item.json.params.lvl1 == 'foo'` → output 0
  - Fallback output (“extra”) → invalid route
- **Connections:**
  - Output 0 → `/foo`
  - Fallback (“extra”) → `Invalid Route`
- **Edge cases:**
  - If `params.lvl1` is missing (e.g., wrong webhook depth), condition is false and falls back.

#### Node: `/foo`
- **Type / role:** Switch; routes second-level segment `lvl2` under `/foo`.
- **Config:**
  - Condition 1: `lvl2 == 'bar'` → output 0
  - Condition 2: `lvl2 == 'qux'` → output 1
  - Fallback → invalid route
- **Connections:**
  - Output 0 → `/foo/bar`
  - Output 1 → `/foo/qux`
  - Fallback (“extra”) → `Invalid Route1`
- **Edge cases:**
  - Requests to `/api/v1/foo` (depth 1) will not have `lvl2`; will fall back unless you create a handler for missing segments.

#### Node: `/foo/bar`
- **Type / role:** Switch; routes third-level segment `lvl3` under `/foo/bar`.
- **Config:**
  - Condition: `lvl3 == 'baz'` → output 0
  - Fallback (“extra”) → invalid route
- **Connections:**
  - Output 0 → `/foo/bar/baz`
  - Fallback → `Invalid Route2`

#### Node: `/foo/bar/baz`
- **Type / role:** NoOp; marks the endpoint match.
- **Connections:** → `baz METHOD`

#### Node: `/foo/qux`
- **Type / role:** Switch placeholder.
- **Config:** Contains a dummy rule (empty equals empty) but effectively still has a fallback output configured as “extra”.
- **Connections:** Output 0 → `Invalid Route3` (so this endpoint is not implemented; it always routes to invalid)
- **Edge cases:**
  - As configured, `/foo/qux` does not have a real implementation route; all requests end up invalid.

#### Node: `Invalid Route` / `Invalid Route1` / `Invalid Route2` / `Invalid Route3`
- **Type / role:** Respond to Webhook; returns a default 400 error.
- **Config:**
  - Response code: `$(' _CFG').item.json.defaults.invalid_route.status_code` (400)
  - Response body (text): `$(' _CFG').item.json.defaults.invalid_route.response`
- **Connections:** Terminal (responds to webhook).
- **Edge cases:**
  - If `_CFG` failed earlier, these expressions will fail.
- **Why duplicates:** Per sticky note, duplicates reduce visual clutter; you could centralize them if preferred.

**Sticky note coverage**
- “Endpoint-based routing … Duplicated fallback nodes to prevent visual clutter…”

---

### 2.4 Method Routing & Endpoint Implementation (placeholders)

**Overview**
Once an endpoint is matched (`/foo/bar/baz`), a Switch routes by HTTP method (GET vs POST). Each branch calls a placeholder sub-workflow node (Execute Workflow) and finally responds with JSON (currently echoing `_REQUEST`).

**Nodes involved**
- `baz METHOD`
- `Check baz status`
- `Create new baz`
- `Implementation`
- `Implementation1`
- `Respond to Webhook1`
- `Respond to Webhook`

#### Node: `baz METHOD`
- **Type / role:** Switch; routes per HTTP method for the `/foo/bar/baz` endpoint.
- **Config (interpreted):**
  - Output “GET” when `$(' _REQUEST').item.json.method == 'GET'`
  - Output “POST” when `$(' _REQUEST').item.json.method == 'POST'`
  - Fallback: “none” (i.e., if PUT/DELETE, nothing happens)
- **Connections:**
  - GET → `Check baz status`
  - POST → `Create new baz`
- **Edge cases:**
  - PUT/DELETE will produce no output (no response). In production you should add a fallback responder (e.g., 405 Method Not Allowed).

#### Node: `Check baz status`
- **Type / role:** NoOp; placeholder for GET handler.
- **Connections:** → `Implementation`

#### Node: `Create new baz`
- **Type / role:** NoOp; placeholder for POST handler.
- **Connections:** → `Implementation1`

#### Node: `Implementation`
- **Type / role:** Execute Workflow; placeholder for real logic implementation (sub-workflow).
- **Config:**
  - Uses “workflowJson: {}” (empty placeholder), “waitForSubWorkflow: false”
  - On error: `continueRegularOutput` (workflow continues even if sub-workflow fails)
- **Connections:** → `Respond to Webhook1`
- **Sub-workflow reference:** None actually configured; you must replace with a real sub-workflow or inline nodes.
- **Edge cases:**
  - As-is, it does nothing meaningful.
  - If replaced with a real sub-workflow and `waitForSubWorkflow=false`, you cannot depend on its outputs in the next node.

#### Node: `Implementation1`
- **Type / role:** Execute Workflow; placeholder for POST logic.
- **Config:** Same as `Implementation`
- **Connections:** → `Respond to Webhook`

#### Node: `Respond to Webhook1`
- **Type / role:** Respond to Webhook; returns JSON.
- **Config:** Respond with JSON body: `{{ $('_REQUEST').item.json }}`
- **Connections:** Terminal.
- **Edge cases:** If `_REQUEST` missing, expression fails.

#### Node: `Respond to Webhook`
- **Type / role:** Respond to Webhook; returns JSON.
- **Config:** Same as `Respond to Webhook1`
- **Connections:** Terminal.

**Sticky note coverage**
- “Endpoint Implementation: Subworkflows are used as a placeholder, implementation can ofc also just be inline.”

---

### 2.5 Examples: $json vs $('NODE') notation (manual section)

**Overview**
A separate manual-triggered set of examples shows (1) how `$json` breaks when a node does not pass data through, (2) how absolute references `$('Named node')...` avoid merges, and (3) how to check `isExecuted` when a node might not run.

**Nodes involved**
- `run examples`
- `example: default $json-based`, `regular Set node`, `non-passthrough node`, `Merge`, `data via $json`
- `example: $('NODE') based`, `Named node`, `non-passthrough node1`, `data via $('NODE')`
- `example: check for execution`, `50:50 randomizer`, `optional path`, `non-passthrough node2`, `data with fallback`

#### Node: `run examples`
- **Type / role:** Manual Trigger; starts the example flows.
- **Connections:** Fan-out to three NoOps:
  - `example: default $json-based`
  - `example: $('NODE') based`
  - `example: check for execution`

#### Node: `example: default $json-based`
- **Type / role:** NoOp; marker for example 1.
- **Connections:** → `regular Set node`

#### Node: `regular Set node`
- **Type / role:** Set node; puts `test = "we need this value"` into `$json`.
- **Connections:** Two parallel paths:
  - → `Merge` (branch 0)
  - → `non-passthrough node`

#### Node: `non-passthrough node`
- **Type / role:** Execute Workflow; represents a node that does not pass through original `$json`.
- **Config:** Placeholder empty sub-workflow JSON; `waitForSubWorkflow=false`, `continueRegularOutput`.
- **Connections:** → `Merge` (branch 1)
- **Edge cases:** Demonstrates that you often need merges/branches to preserve `$json` context.

#### Node: `Merge`
- **Type / role:** Merge; `chooseBranch` mode to pick one branch’s data.
- **Connections:** → `data via $json`
- **Edge cases:** In real workflows, merges become visually complex as branches multiply.

#### Node: `data via $json`
- **Type / role:** Set node; attempts to read `={{ $json.test }}`.
- **Key point:** If the chosen branch lacks `test`, this becomes undefined—illustrating the problem.

---

#### Node: `example: $('NODE') based`
- **Type / role:** NoOp; marker for example 2.
- **Connections:** → `Named node`

#### Node: `Named node`
- **Type / role:** Set node; sets `test = "we need this value"`.
- **Connections:** → `non-passthrough node1`

#### Node: `non-passthrough node1`
- **Type / role:** Execute Workflow; non–pass-through placeholder.
- **Connections:** → `data via $('NODE')`

#### Node: `data via $('NODE')`
- **Type / role:** Set node; reads via absolute reference:
  - `={{ $('Named node').item.json.test }}`
- **Key advantage:** Works even if intermediate nodes don’t pass through data (as long as the referenced node is on the executed path).

---

#### Node: `example: check for execution`
- **Type / role:** NoOp; marker for example 3.
- **Connections:** → `50:50 randomizer`

#### Node: `50:50 randomizer`
- **Type / role:** IF node; randomly chooses a path.
- **Config:** Condition `Math.random() >= 0.5`
- **Connections:**
  - True → `optional path`
  - False → `non-passthrough node2`

#### Node: `optional path`
- **Type / role:** Set node; sets `test = "we need this value"`.
- **Connections:** → `non-passthrough node2`

#### Node: `non-passthrough node2`
- **Type / role:** Execute Workflow placeholder.
- **Connections:** → `data with fallback`

#### Node: `data with fallback`
- **Type / role:** Set node; safe access based on execution:
  - `={{ $('optional path').isExecuted ? $('optional path').item.json.test : 'node not executed' }}`
- **Edge cases:** If you rename `optional path`, n8n updates references automatically.

**Sticky note coverage**
- Explains why absolute references reduce branching/merging, and the rule: referenced node must be on the path and must have executed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| v1/seg1 | Webhook | API entry (1 segment) | — | -> GET, -> POST, -> PUT, -> DELETE | ## API routing levels  \n3 levels provide a solid base\n### To add more:\n- Copy the highest-level Webhook-node\n- Add another variable to the end of the path: `/:lvl4`\n- Connect the outputs to the corresponding mode Set-node |
| v1/seg2 | Webhook | API entry (2 segments) | — | -> GET, -> POST, -> PUT, -> DELETE | ## API routing levels  \n3 levels provide a solid base\n### To add more:\n- Copy the highest-level Webhook-node\n- Add another variable to the end of the path: `/:lvl4`\n- Connect the outputs to the corresponding mode Set-node |
| v1/seg3 | Webhook | API entry (3 segments) | — | -> GET, -> POST, -> PUT, -> DELETE | ## API routing levels  \n3 levels provide a solid base\n### To add more:\n- Copy the highest-level Webhook-node\n- Add another variable to the end of the path: `/:lvl4`\n- Connect the outputs to the corresponding mode Set-node |
| -> GET | Set | Normalize HTTP method | v1/seg1, v1/seg2, v1/seg3 | API entry point | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| -> POST | Set | Normalize HTTP method | v1/seg1, v1/seg2, v1/seg3 | API entry point | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| -> PUT | Set | Normalize HTTP method | v1/seg1, v1/seg2, v1/seg3 | API entry point | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| -> DELETE | Set | Normalize HTTP method | v1/seg1, v1/seg2, v1/seg3 | API entry point | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| API entry point | NoOp | Convergence point | -> GET, -> POST, -> PUT, -> DELETE | _REQUEST | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| _REQUEST | Set | Store request context (global) | API entry point | _CFG | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| _CFG | Code | Global config + base URL extraction | _REQUEST | Clear JSON | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| Clear JSON | Set | Clear `$json` for clean flow | _CFG | Routing | ## Streamlining data  \nWe initialize global data for the _REQUEST, and set up our _CFG. We then clear the JSON, so we have a clean separation of global data vs. data flow. |
| Routing | NoOp | Start routing section | Clear JSON | api root | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| api root | Switch | Route by lvl1 | Routing | /foo, Invalid Route | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| /foo | Switch | Route by lvl2 | api root | /foo/bar, /foo/qux, Invalid Route1 | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| /foo/bar | Switch | Route by lvl3 | /foo | /foo/bar/baz, Invalid Route2 | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| /foo/qux | Switch | Placeholder route (not implemented) | /foo | Invalid Route3 | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| /foo/bar/baz | NoOp | Endpoint marker | /foo/bar | baz METHOD | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| baz METHOD | Switch | Route /baz by HTTP method | /foo/bar/baz | Check baz status, Create new baz | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Check baz status | NoOp | GET handler placeholder | baz METHOD | Implementation | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Create new baz | NoOp | POST handler placeholder | baz METHOD | Implementation1 | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Implementation | Execute Workflow | Placeholder implementation (GET) | Check baz status | Respond to Webhook1 | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Implementation1 | Execute Workflow | Placeholder implementation (POST) | Create new baz | Respond to Webhook | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Respond to Webhook1 | Respond to Webhook | Return JSON response (GET) | Implementation | — | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Respond to Webhook | Respond to Webhook | Return JSON response (POST) | Implementation1 | — | ## Endpoint Implementation\nSubworkflows are used as a placeholder, implementation can ofc also just be inline. |
| Invalid Route | Respond to Webhook | Default invalid route response | api root | — | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| Invalid Route1 | Respond to Webhook | Default invalid route response | /foo | — | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| Invalid Route2 | Respond to Webhook | Default invalid route response | /foo/bar | — | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| Invalid Route3 | Respond to Webhook | Default invalid route response | /foo/qux | — | ## Endpoint-based routing\nDuplicated fallback nodes to prevent visual clutter, all fallbacks can ofc also route to a centralized flow. |
| run examples | Manual Trigger | Start example section | — | example: default $json-based; example: $('NODE') based; example: check for execution | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| example: default $json-based | NoOp | Example marker | run examples | regular Set node | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| regular Set node | Set | Put test value into `$json` | example: default $json-based | Merge; non-passthrough node | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| non-passthrough node | Execute Workflow | Simulate non-pass-through | regular Set node | Merge | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| Merge | Merge | ChooseBranch merge for `$json` example | regular Set node; non-passthrough node | data via $json | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| data via $json | Set | Read via `$json.test` | Merge | — | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| example: $('NODE') based | NoOp | Example marker | run examples | Named node | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| Named node | Set | Put test value into named node | example: $('NODE') based | non-passthrough node1 | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| non-passthrough node1 | Execute Workflow | Simulate non-pass-through | Named node | data via $('NODE') | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| data via $('NODE') | Set | Read via `$('Named node')...` | non-passthrough node1 | — | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| example: check for execution | NoOp | Example marker | run examples | 50:50 randomizer | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| 50:50 randomizer | If | Randomly execute optional node | example: check for execution | optional path; non-passthrough node2 | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| optional path | Set | Optional data producer | 50:50 randomizer | non-passthrough node2 | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| non-passthrough node2 | Execute Workflow | Simulate non-pass-through | 50:50 randomizer / optional path | data with fallback | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| data with fallback | Set | Read with `isExecuted` fallback | non-passthrough node2 | — | ## Example for $json-based vs. absolute $('_NODE') notation for data access\nNodes that don't pass-through content... |
| Sticky Note | Sticky Note | Comment | — | — |  |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note5 | Sticky Note | Comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create 3 Webhook nodes** (for 1–3 path depths)
   1) Add **Webhook** node `v1/seg1`
   - Path: `/api/v1/:lvl1`
   - Methods: enable GET, POST, PUT, DELETE (multiple methods)
   - Response Mode: **Using “Respond to Webhook” node**
   2) Duplicate to create `v1/seg2` with path `/api/v1/:lvl1/:lvl2`
   3) Duplicate to create `v1/seg3` with path `/api/v1/:lvl1/:lvl2/:lvl3`

2. **Create 4 Set nodes to normalize the method**
   - Add Set node `-> GET`: set field `method` to `"GET"`, enable pass-through (include other fields).
   - Add Set node `-> POST`: `method = "POST"`, pass-through.
   - Add Set node `-> PUT`: `method = "PUT"`, pass-through.
   - Add Set node `-> DELETE`: `method = "DELETE"`, pass-through.

3. **Connect each Webhook’s method outputs to the method Set nodes**
   - For each `v1/segX`, connect:
     - GET output → `-> GET`
     - POST output → `-> POST`
     - PUT output → `-> PUT`
     - DELETE output → `-> DELETE`

4. **Create a NoOp convergence node**
   - Add **NoOp** node `API entry point`
   - Connect all `-> GET/POST/PUT/DELETE` → `API entry point`

5. **Create `_REQUEST` global request capture**
   - Add **Set** node `_REQUEST`
   - Configure to include everything **except** `headers` (exclude field `headers`)
   - Connect `API entry point` → `_REQUEST`

6. **Create `_CFG` Code node**
   - Add **Code** node `_CFG`
   - Paste logic that:
     - Builds config object with defaults and `api_base_url: 'auto'`
     - Derives `api_base_url` from `$(' _REQUEST').item.json.webhookUrl` by extracting `/api/v{version}`
   - Connect `_REQUEST` → `_CFG`

7. **Add “Clear JSON” node**
   - Add **Set** node `Clear JSON` with no assignments (empty object output is fine)
   - Enable **Always Output Data**
   - Connect `_CFG` → `Clear JSON`

8. **Create routing skeleton**
   1) Add **NoOp** node `Routing`; connect `Clear JSON` → `Routing`
   2) Add **Switch** node `api root`
      - Rule: `$(' _REQUEST').item.json.params.lvl1 == 'foo'`
      - Fallback output enabled (“extra”)
      - Connect `Routing` → `api root`
   3) Add **Switch** node `/foo`
      - Rule 1: `lvl2 == 'bar'` via expression on `_REQUEST`
      - Rule 2: `lvl2 == 'qux'`
      - Fallback output enabled
      - Connect `api root (match)` → `/foo`

9. **Add invalid route responders (duplicated or centralized)**
   - Add **Respond to Webhook** node `Invalid Route`
     - Respond With: Text
     - Response Code: `={{ $('_CFG').item.json.defaults.invalid_route.status_code }}`
     - Body: `={{ $('_CFG').item.json.defaults.invalid_route.response }}`
   - Duplicate to create `Invalid Route1`, `Invalid Route2`, `Invalid Route3`
   - Connect:
     - `api root (fallback)` → `Invalid Route`
     - `/foo (fallback)` → `Invalid Route1`

10. **Add third-level routing under `/foo/bar`**
   - Add **Switch** node `/foo/bar`
     - Rule: `$(' _REQUEST').item.json.params.lvl3 == 'baz'`
     - Fallback enabled
   - Connect `/foo (bar branch)` → `/foo/bar`
   - Connect `/foo/bar (fallback)` → `Invalid Route2`
   - Add **NoOp** node `/foo/bar/baz`
   - Connect `/foo/bar (match)` → `/foo/bar/baz`

11. **Add method router for `/foo/bar/baz`**
   - Add **Switch** node `baz METHOD`
     - Output “GET” when `_REQUEST.method == 'GET'`
     - Output “POST” when `_REQUEST.method == 'POST'`
     - Set fallback output to “none” (or improve with a 405 responder)
   - Connect `/foo/bar/baz` → `baz METHOD`

12. **Add placeholder handlers + sub-workflow placeholders**
   - Add **NoOp** node `Check baz status` (GET)
   - Add **NoOp** node `Create new baz` (POST)
   - Connect `baz METHOD (GET)` → `Check baz status`
   - Connect `baz METHOD (POST)` → `Create new baz`
   - Add **Execute Workflow** node `Implementation`
     - Wait for Sub-workflow: **false**
     - Error handling: **Continue (continueRegularOutput)**
     - Replace with a real sub-workflow selection (recommended) or implement inline
   - Add **Execute Workflow** node `Implementation1` with same settings
   - Connect:
     - `Check baz status` → `Implementation`
     - `Create new baz` → `Implementation1`

13. **Add final responders**
   - Add **Respond to Webhook** node `Respond to Webhook1`
     - Respond with: JSON
     - Body: `={{ $('_REQUEST').item.json }}`
   - Add **Respond to Webhook** node `Respond to Webhook`
     - Same JSON body expression
   - Connect:
     - `Implementation` → `Respond to Webhook1`
     - `Implementation1` → `Respond to Webhook`

14. **(Optional) Add the example section**
   - Add `run examples` (Manual Trigger) and recreate the three demonstration flows exactly:
     - `$json` flow: NoOp → Set → Execute Workflow → Merge(chooseBranch) → Set reading `$json.test`
     - `$('NODE')` flow: NoOp → Set (“Named node”) → Execute Workflow → Set reading `$('Named node').item.json.test`
     - `isExecuted` flow: NoOp → IF random → optional Set → Execute Workflow → Set using `$('optional path').isExecuted ? ... : ...`

**Credentials**
- None required in this template as provided (no external services configured).
- If you replace Execute Workflow placeholders with real integrations, add the relevant credentials in those nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Webhooks with path variables automatically add a hash as URL base; and each path depth needs its own Webhook node (as of n8n v2.3.4). | Sticky notes (workflow constraints) |
| Global/absolute node references: use `$('NODE_NAME').item.json` to reduce branching/merging when nodes don’t pass through data. | Sticky note “Direct node notation & ‘global’ naming” |
| To reference a node absolutely: it must be on the execution path and must have executed; use `isExecuted` for optional branches. | Sticky note “check for execution” guidance |
| Template can serve JSON APIs or HTML responses; suitable for fast early prototyping. | Sticky note “What it can be used for” |

Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.