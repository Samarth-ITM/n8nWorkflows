Generate product and marketing images using Riverflow 2.0 on Replicate

https://n8nworkflows.xyz/workflows/generate-product-and-marketing-images-using-riverflow-2-0-on-replicate-13303


# Generate product and marketing images using Riverflow 2.0 on Replicate

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This n8n solution generates and retrieves **product/marketing images** using **Riverflow 2.0 (fast/pro) on Replicate**. It collects parameters via an n8n Form, sanitizes/normalizes them to match Replicate’s expected payload, triggers image generation jobs in parallel (when multiple outputs are requested), and then **polls for completion** using a shared **Data Table** to aggregate results from sub-workflow executions. Finally, it returns the completed image URLs and optionally downloads the raw images.

**Target use cases:**
- Editing or generating marketing visuals from an initial image with text changes (e.g., “make the text on the shampoo bottle say Riverflow”)
- Landing page mockups and product creative variations
- Batch generation (up to 4 outputs) with parallel processing

**Main logical blocks (parent workflow + sub-workflow embedded in same JSON):**
1.1 **Input Reception (Form Trigger)**  
1.2 **Input Sanitization & Fan-out (Code)**  
1.3 **Parallel Generation via Sub-workflow (Execute Workflow)**  
1.4 **Process Tracking via Data Table (Insert “process” row)**  
1.5 **Polling Until All Outputs Arrive (Wait + Data Table get + If loop)**  
1.6 **Output Consolidation & Image Fetching (Filter URLs + HTTP download)**  
2.x **Sub-workflow: Replicate POST + GET polling + Data Table write-back**

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception (Form Trigger)

**Overview:**  
Collects all Riverflow parameters from the user through an n8n hosted form and starts the parent workflow.

**Nodes involved:**
- **On form submission** (Form Trigger)

**Node details**
- **On form submission**
  - **Type / role:** `n8n-nodes-base.formTrigger` — entry point; collects user input.
  - **Key configuration choices:**
    - Form title: *Input data for Riverflow 2.0 Image Generation*
    - Description: instructs comma-separated values for multi-value fields.
    - Fields include:
      - `Model` dropdown: `pro` or `fast`
      - `Font Urls`, `Font Texts` (multi-value supported via commas)
      - `Resolution` dropdown: `1K` (default) or `2K`
      - `Init Images` (multi-value supported via commas; used for image editing or conditioning)
      - `Instruction` textarea (**required**)
      - `Aspect Ratio` dropdown: default `auto` + many ratios
      - Checkboxes: `Transparency`, `Enhance Prompt`, `Safety Checker` (default true)
      - `Max Iterations` number default 1 (capped later by code)
      - `Super Resolution Refs` (multi-value supported via commas)
      - `Number of Outputs` number default 1 (capped later by code)
  - **Outputs:** to **Parse inputs given from form**
  - **Edge cases / failures:**
    - Missing required `Instruction` prevents submission.
    - Checkbox values arrive in n8n “checkbox array-style” (handled later).
    - Comma-separated multi-values can include empty tokens/spaces (handled later).
  - **Version notes:** TypeVersion `2.5` (Form Trigger).

**Sticky notes relevant (context):**
- “### Receiving and handling input” (covers this block’s area)
- Main overview sticky note includes setup guidance (Replicate key + Data Table).

---

### 1.2 Input Sanitization & Fan-out

**Overview:**  
Normalizes field names, validates constraints, converts types (arrays/booleans), selects the correct Replicate model endpoint, generates a shared `processID`, and duplicates items to enable parallel jobs for multiple outputs.

**Nodes involved:**
- **Parse inputs given from form** (Code)

**Node details**
- **Parse inputs given from form**
  - **Type / role:** `n8n-nodes-base.code` — transforms the form payload into a clean internal schema.
  - **Key configuration choices (interpreted):**
    - Normalizes keys: lowercase + spaces → underscores (e.g., `Number of Outputs` → `number_of_outputs`)
    - Removes field: `formmode`
    - Converts these to booleans: `transparency`, `enhance_prompt`, `safety_checker`  
      - Checkbox “true” becomes `true`, anything else becomes `false`
    - Converts these to arrays (splitting comma-separated strings):
      - `init_images`, `font_urls`, `font_texts`, `super_resolution_refs`
    - Validations:
      - `max_iterations` must be `<= 3` (throws error if exceeded)
      - `number_of_outputs` must be `<= 4`
      - `font_urls` max length `2`
      - `init_images` max length depends on model:
        - `fast`: max `4`
        - `pro`: max `10`
      - `super_resolution_refs` max length `4`
    - Sets Replicate prediction endpoint:
      - if `Model === "pro"` → `https://api.replicate.com/v1/models/sourceful/riverflow-2.0-pro/predictions`
      - else → `https://api.replicate.com/v1/models/sourceful/riverflow-2.0-fast/predictions`
    - Generates one UUID per form submission: `processID = crypto.randomUUID()`
    - **Fan-out:** duplicates the sanitized item `count = number_of_outputs` times, so the downstream ExecuteWorkflow runs once per desired output.
  - **Outputs:** to **Call 'POST + GET requests sub-workflow'** (one item per requested output)
  - **Edge cases / failures:**
    - Any validation violation throws, stopping execution (good safety, but user-visible failure).
    - If `number_of_outputs` is missing/empty, defaults to 1.
    - If checkboxes return unexpected structures, boolean conversion may default to `false`.
  - **Version notes:** TypeVersion `2` (Code node).

**Sticky notes relevant:**
- “### Receiving and handling input”

---

### 1.3 Parallel Generation via Sub-workflow (Execute Workflow)

**Overview:**  
Invokes a separate workflow that performs the Replicate POST call and GET polling, then writes results into a shared Data Table.

**Nodes involved:**
- **Call 'POST + GET requests sub-workflow'** (Execute Workflow)

**Node details**
- **Call 'POST + GET requests sub-workflow'**
  - **Type / role:** `n8n-nodes-base.executeWorkflow` — launches the sub-workflow once per input item (parallel-capable).
  - **Execution mode:** `each` (one sub-workflow execution per input item).
  - **Wait behavior:** `waitForSubWorkflow: false`  
    - Parent continues immediately; results are retrieved indirectly through Data Table polling.
  - **Sub-workflow referenced:**  
    - Workflow name: **HTTPs Requests for Riverflow 2.0 Image Generation**
    - Workflow ID: `s5aseRc6thQFXxv4`
  - **Inputs mapping (key expressions):**
    - `url = {{$json.url}}`
    - `process_id = {{$json.processID}}`
    - boolean/number fields passed directly: `transparency`, `enhance_prompt`, `max_iterations`, `safety_checker`
    - many fields passed as “*_toJsonString” values, then later `.toJsonString()` is used in HTTP body:
      - `font_urls_toJsonString = {{$json.font_urls}}` (array)
      - `font_texts_toJsonString = {{$json.font_texts}}`
      - `resolution_toJsonString = {{$json.resolution}}` (string)
      - `init_images_toJsonString = {{$json.init_images}}`
      - `instruction_toJsonString = {{$json.instruction}}` (string)
      - `aspect_ratio_toJsonString = {{$json.aspect_ratio}}` (string)
      - `super_resolution_refs_toJsonString = {{$json.super_resolution_refs}}`
  - **Outputs:** to **Isolate 1 process id**
  - **Edge cases / failures:**
    - If the sub-workflow is missing/disabled or ID differs, executions fail.
    - If `waitForSubWorkflow=false`, parent won’t see direct errors from sub-flow unless separately logged; failures can manifest as “never completes” in polling.
    - Incorrect input typing can break `.toJsonString()` usage in the sub-workflow POST body if values are not present.
  - **Version notes:** TypeVersion `1.3` (Execute Workflow node).

**Sticky notes relevant:**
- “### Sub-Flow … POST request … GET request … output url returned.”

---

### 1.4 Process Tracking via Data Table (Insert placeholder row)

**Overview:**  
Stores a tracking row keyed by `process` (UUID) with the expected number of outputs so the parent workflow can poll for completion.

**Nodes involved:**
- **Isolate 1 process id** (Code)
- **Insert empty process id** (Data Table)
- **Isolate 1 process id1** (Code)

**Node details**
- **Isolate 1 process id**
  - **Type / role:** Code — takes only the first item from the fan-out list.
  - **Why needed:** the parent only needs one tracking record per submission, not one per output.
  - **Outputs:** to **Insert empty process id**
  - **Edge cases:** if input is empty returns `[]` and the workflow stops silently.
- **Insert empty process id**
  - **Type / role:** `n8n-nodes-base.dataTable` — inserts a row into Data Table **Processes**.
  - **Operation:** “insert” (inferred from using `columns.value` without `operation:get`)
  - **Data written:**
    - `process = {{$input.first().json.process_id}}`  
      (note: in parent items the field is `processID`; however this node expects `process_id`. This works only if the incoming item actually contains `process_id`. See edge case below.)
    - `processcount = {{$input.first().json.number_of_outputs}}`
    - Other schema fields present in table: `time`, `status`, `output` (left blank here)
  - **Data Table:** `Processes` (ID `SdzSGfEAOeEmquQm`)
  - **Outputs:** to **Isolate 1 process id1**
  - **Edge cases / likely integration pitfall:**
    - **Field name mismatch risk:** parent normalization sets `processID` (capital D), while this Data Table insert references `process_id`.  
      In practice, this can cause `process` to insert as `null/undefined`, breaking polling. If your actual executions work, it likely means the inbound item includes `process_id` from the Execute Workflow output mapping, but this parent workflow does **not** wait for sub-workflow results—so you should verify what the Execute Workflow node outputs immediately.
    - Data Table missing or wrong schema causes insert failure.
- **Isolate 1 process id1**
  - **Type / role:** Code — returns only the first item (again) to ensure a single control item for polling.
  - **Outputs:** to **Wait**
  - **Edge cases:** empty input returns `[]`.

**Sticky notes relevant:**
- “### Polling process id till all complete”
- “### Data Table … communicate between sub-workflows and parent”

---

### 1.5 Polling Until All Outputs Arrive

**Overview:**  
Waits, then queries the Data Table for all rows with the matching `process`. Repeats until the number of returned rows indicates all outputs have completed.

**Nodes involved:**
- **Wait** (Wait)
- **Get how many processes done** (Data Table get)
- **If number of processes is correct** (If)

**Node details**
- **Wait**
  - **Type / role:** `n8n-nodes-base.wait` — delays polling.
  - **Config:** `amount: 10` (seconds by default for Wait node)
  - **Outputs:** to **Get how many processes done**
- **Get how many processes done**
  - **Type / role:** Data Table “get” — fetches all rows where `process == {{$input.first().json.process}}`
  - **Operation:** `get`
  - **Outputs:** to **If number of processes is correct**
  - **Edge cases:**
    - If `process` is missing/incorrect, query returns nothing; loop never completes.
- **If number of processes is correct**
  - **Type / role:** If — controls loop termination.
  - **Condition implemented:**
    - `left = {{$input.all().length}}`
    - `right = {{$input.first().json.processcount + 1}}`
    - Operator: `notEquals`
  - **Interpretation:**  
    - The workflow expects **(processcount + 1)** rows to exist before completion. The “+1” corresponds to the initial placeholder row inserted by the parent workflow.
  - **True branch (not equals):** loops back via **Isolate 1 process id1** → **Wait** (continues polling)
  - **False branch (equals):** goes to **Output Filtering**
  - **Edge cases:**
    - If sub-workflow writes rows but does not include the placeholder logic, the +1 logic can be wrong.
    - If any sub-workflow run fails and never writes a row, polling loops indefinitely (no max attempts / timeout in parent).

**Sticky notes relevant:**
- “### Polling process id till all complete”

---

### 1.6 Output Consolidation & Image Fetching

**Overview:**  
Extracts output URLs from Data Table records, removes duplicates, and downloads image binaries.

**Nodes involved:**
- **Output Filtering** (Code)
- **Get image/s** (HTTP Request)

**Node details**
- **Output Filtering**
  - **Type / role:** Code — selects non-empty `output` fields, deduplicates them, emits `{ url }` items.
  - **Logic highlights:**
    - Collect `item.json.output` if it’s a non-empty string
    - `uniqueOutputs = [...new Set(outputUrls)]`
    - Returns `[{json:{url}}...]`
  - **Outputs:** to **Get image/s**
  - **Edge cases:**
    - If the Data Table row stores output under a different field name, nothing is returned.
- **Get image/s**
  - **Type / role:** `n8n-nodes-base.httpRequest` — downloads the image from each URL.
  - **Config:** `url = {{$json.url}}`
  - **Outputs:** (final output of parent workflow execution)
  - **Edge cases:**
    - Replicate delivery URLs may expire; downloading later may fail.
    - Large images can exceed memory/time limits depending on n8n settings.

**Sticky notes relevant:**
- “### Outputing Images”

---

## 2.7 Sub-workflow: HTTPs Requests for Riverflow 2.0 through Replicate (embedded nodes)

> This JSON includes the sub-workflow nodes as well. In a real n8n instance, these nodes live in the sub-workflow referenced by the parent’s Execute Workflow node.

### Sub-block A: Sub-workflow Entry & POST to Replicate

**Overview:**  
Receives parameters from the parent, triggers a Replicate prediction via POST, then begins polling the returned `urls.get`.

**Nodes involved:**
- **Start** (Execute Workflow Trigger)
- **POST request to Replicate API** (HTTP Request)
- **Wait1** (Wait)

**Node details**
- **Start**
  - **Type / role:** `executeWorkflowTrigger` — entry point for called workflows.
  - **Inputs defined:** `url`, `process_id`, plus all “*_toJsonString” fields and booleans/numbers.
  - **Connections:** outputs to both **POST request to Replicate API** and **Merge** (to pass `process_id` alongside results)
- **POST request to Replicate API**
  - **Type / role:** HTTP POST — creates a prediction.
  - **Authentication:** `httpBearerAuth` credential **Replicate API key**
  - **Headers:** `Prefer: wait` (Replicate may wait briefly before responding)
  - **Body:** JSON with `input`:
    - Uses `{{ ...toJsonString() }}` for arrays/strings to ensure proper JSON quoting/format.
    - Passes booleans and numeric fields directly.
  - **Output:** to **Wait1**
  - **Failure modes:**
    - 401/403 if API key invalid
    - 422 if payload invalid (wrong types, unsupported aspect_ratio, too many init_images, etc.)
    - Model endpoint mismatch (fast vs pro) if `url` is wrong
- **Wait1**
  - **Type / role:** Wait — short delay between POST and GET polling
  - **Config:** `amount: 2`
  - **Output:** to **GET Request for Riverflow Predictions**

**Sticky notes relevant:**
- “### Riverflow 2.0 call”
- “## Sub-Workflow: HTTPs Requests for RIverflow 2.0 through Replicate”

---

### Sub-block B: GET Poll Loop Until Complete

**Overview:**  
Repeatedly calls the prediction `urls.get` until status is no longer “processing”, then emits a simplified result.

**Nodes involved:**
- **GET Request for Riverflow Predictions** (HTTP Request)
- **Check if output is present** (If)
- **Return output image url** (Set)
- **Wait1** (Wait) [loop back]

**Node details**
- **GET Request for Riverflow Predictions**
  - **Type / role:** HTTP GET — fetches prediction status/output.
  - **URL:** `{{$json.urls.get}}` (from Replicate POST response)
  - **Timeout:** 6000 ms
  - **Auth:** Replicate bearer token
  - **Outputs:** to **Check if output is present**
  - **Failure modes:** transient network errors, 429 rate limits, timeouts.
- **Check if output is present**
  - **Type / role:** If — decides whether to wait and recheck.
  - **Condition:** if `{{$json.status}} == "processing"` then **True** branch to **Wait1** (continue polling)  
    Else **False** branch to **Return output image url**
  - **Edge cases:**
    - Replicate can return statuses like `starting`, `succeeded`, `failed`, `canceled`. This logic only loops on exactly `processing`; it will stop on `starting` (likely too early). Consider including `starting` in loop condition.
- **Return output image url**
  - **Type / role:** Set — maps Replicate response into simplified fields:
    - `output` (array) = `{{$json.output}}`
    - `time` (string) = `{{$json.created_at}}`
    - `status` (string) = `{{$json.status}}`
  - **Output:** to **Merge** input 1

**Sticky notes relevant:**
- “### Loop to check get request till status is no longer processing”

---

### Sub-block C: Merge process_id + results, then write to Data Table

**Overview:**  
Merges the original `process_id` with the finished Replicate result, formats it into one row per generated output, and inserts into the shared Data Table so the parent can detect completion.

**Nodes involved:**
- **Merge** (Merge)
- **Code in JavaScript** (Code)
- **Insert row** (Data Table)

**Node details**
- **Merge**
  - **Type / role:** Merge — combines:
    - Input 0: from **Start** (contains `process_id`)
    - Input 1: from **Return output image url** (contains Replicate result)
  - **Mode:** default (n8n default merge behavior; typically “append” unless configured). Here it’s used as a simple way to ensure both items are available downstream.
- **Code in JavaScript**
  - **Type / role:** Code — constructs clean rows for Data Table.
  - **Logic highlights:**
    - Finds `process_id` from any item that has it.
    - Keeps only items where `item.json.output` is an array (Replicate result).
    - Emits objects:
      - `process_id`
      - `status`
      - `time`
      - `output_url = item.json.output[0]` (takes first output from array)
  - **Edge cases:**
    - If Replicate returns multiple URLs in `output`, only the first is stored.
    - If prediction fails and `output` is null, it will be filtered out and nothing is inserted → parent will poll forever.
- **Insert row**
  - **Type / role:** Data Table insert — writes completion row.
  - **Data Table:** `Processes` (same ID)
  - **Columns written:**
    - `process = {{$json.process_id}}`
    - `time = {{$json.time}}`
    - `status = {{$json.status}}`
    - `output = {{$json.output_url}}`
  - **Failure modes:** Data Table schema mismatch, permission issues.

**Sticky notes relevant:**
- “### Gather parameters needed to add to data table”
- “### Insert into data table for parent workflow”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Collect user parameters and start parent flow | — | Parse inputs given from form | ### Receiving and handling input |
| Parse inputs given from form | Code | Normalize/validate inputs, choose model URL, fan-out N outputs | On form submission | Call 'POST + GET requests sub-workflow' | ### Receiving and handling input |
| Call 'POST + GET requests sub-workflow' | Execute Workflow | Launch sub-workflow in parallel for each output item | Parse inputs given from form | Isolate 1 process id | ### Sub-Flow\nRuns another work-flow which contains nodes to:\n- send POST request to Replicate \n- do a GET request to check for errors and status. \n- If no errors, the output url is returned. |
| Isolate 1 process id | Code | Keep only one item for process tracking | Call 'POST + GET requests sub-workflow' | Insert empty process id | ### Polling process id till all complete |
| Insert empty process id | Data Table | Insert placeholder row with process + expected count | Isolate 1 process id | Isolate 1 process id1 | ### Data Table\nData table used as a way to **communicate between the sub-workflows and the parent workflow** so that we can keep track of if they are complete or not. |
| Isolate 1 process id1 | Code | Ensure a single control item continues to polling loop | Insert empty process id; If number of processes is correct (true branch) | Wait | ### Polling process id till all complete |
| Wait | Wait | Delay between polling attempts | Isolate 1 process id1 | Get how many processes done | ### Polling process id till all complete |
| Get how many processes done | Data Table (get) | Read all rows for this process | Wait | If number of processes is correct | ### Polling process id till all complete |
| If number of processes is correct | If | Loop until expected rows exist, else proceed | Get how many processes done | Isolate 1 process id1 (true); Output Filtering (false) | ### Polling process id till all complete |
| Output Filtering | Code | Extract/dedupe output URLs from Data Table rows | If number of processes is correct (false branch) | Get image/s | ### Outputing Images |
| Get image/s | HTTP Request | Download each output image from URL | Output Filtering | — | ### Outputing Images |
| Start | Execute Workflow Trigger | Entry point for the sub-workflow | — | POST request to Replicate API; Merge | ## Sub-Workflow: HTTPs Requests for RIverflow 2.0 through Replicate |
| POST request to Replicate API | HTTP Request | Create Replicate prediction | Start | Wait1 | ### Riverflow 2.0 call |
| Wait1 | Wait | Delay between GET polling attempts | POST request to Replicate API; Check if output is present (true) | GET Request for Riverflow Predictions | ### Loop to check get request till status is no longer processing |
| GET Request for Riverflow Predictions | HTTP Request | Poll prediction status | Wait1 | Check if output is present | ### Loop to check get request till status is no longer processing |
| Check if output is present | If | Continue polling while status is processing | GET Request for Riverflow Predictions | Wait1 (true); Return output image url (false) | ### Loop to check get request till status is no longer processing |
| Return output image url | Set | Map replicate response to output/time/status | Check if output is present (false) | Merge | ### Loop to check get request till status is no longer processing |
| Merge | Merge | Combine process_id + replicate result | Start; Return output image url | Code in JavaScript | ### Gather parameters needed to add to data table |
| Code in JavaScript | Code | Format Data Table row(s) with process_id and output_url | Merge | Insert row | ### Gather parameters needed to add to data table |
| Insert row | Data Table | Insert completed output record for parent polling | Code in JavaScript | — | ### Insert into data table for parent workflow |
| Sticky Note | Sticky Note | Documentation comment | — | — |  |
| Sticky Note1 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note2 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note3 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note4 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note5 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note6 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note7 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note8 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note9 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note10 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note11 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note12 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note13 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note14 | Sticky Note | Documentation comment | — | — |  |
| Sticky Note15 | Sticky Note | Documentation comment | — | — |  |

> Sticky Note nodes are included to satisfy “do not skip any nodes”; they do not connect to runtime flow.

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1. **Replicate account + API token**: https://replicate.com/  
2. **n8n Data Table** named **Processes** with at least these columns:
   - `process` (string)
   - `time` (dateTime)
   - `status` (string)
   - `output` (string)
   - `processcount` (number)  
3. **HTTP Bearer Auth credential** in n8n:
   - Name: “Replicate API key”
   - Token: your Replicate API token

---

### A) Create the Sub-workflow: “HTTPs Requests for Riverflow 2.0 Image Generation”

1. Create a new workflow named **HTTPs Requests for Riverflow 2.0 Image Generation**.
2. Add node **Execute Workflow Trigger** named **Start**.
   - Define inputs:  
     `url, process_id, transparency, enhance_prompt, max_iterations, safety_checker, font_urls_toJsonString, font_texts_toJsonString, resolution_toJsonString, init_images_toJsonString, instruction_toJsonString, aspect_ratio_toJsonString, super_resolution_refs_toJsonString`
3. Add **HTTP Request** node named **POST request to Replicate API**:
   - Method: **POST**
   - URL: `={{ $json.url }}`
   - Authentication: **Predefined Credential Type** → **HTTP Bearer Auth** → select **Replicate API key**
   - Headers: add `Prefer: wait`
   - Body type: JSON
   - JSON body (structure):
     - `input.font_urls` from `font_urls_toJsonString` (serialized using `.toJsonString()`)
     - `input.font_texts` similarly
     - `input.resolution`, `init_images`, `instruction`, `aspect_ratio`, `super_resolution_refs` similarly
     - `input.transparency`, `enhance_prompt`, `max_iterations`, `safety_checker` directly
4. Add **Wait** node named **Wait1**:
   - Amount: `2` (seconds)
5. Connect: **Start → POST request to Replicate API → Wait1**
6. Add **HTTP Request** node named **GET Request for Riverflow Predictions**:
   - Method: GET (default)
   - URL: `={{ $json.urls.get }}`
   - Timeout option: `6000` ms
   - Authentication: same **Replicate API key**
7. Connect: **Wait1 → GET Request for Riverflow Predictions**
8. Add **If** node named **Check if output is present**:
   - Condition: `={{ $json.status }}` equals `processing`
   - True output: continue polling
   - False output: finished (or failed)
9. Connect: **GET Request → Check if output is present**
10. Connect **True** branch to **Wait1** (loop).
11. Add **Set** node named **Return output image url**:
    - Set fields:
      - `output = {{$json.output}}` (type array)
      - `time = {{$json.created_at}}`
      - `status = {{$json.status}}`
12. Connect **False** branch to **Return output image url**
13. Add **Merge** node named **Merge**.
    - Connect **Start → Merge (input 0)** (to carry `process_id`)
    - Connect **Return output image url → Merge (input 1)**
14. Add **Code** node named **Code in JavaScript**:
    - Extract `process_id`
    - Keep only items where `output` is an array
    - Output: `{process_id, status, time, output_url: output[0]}`
15. Connect: **Merge → Code in JavaScript**
16. Add **Data Table** node named **Insert row**:
    - Operation: Insert (default insert behavior)
    - Data Table: **Processes**
    - Map columns:
      - `process = {{$json.process_id}}`
      - `time = {{$json.time}}`
      - `status = {{$json.status}}`
      - `output = {{$json.output_url}}`
17. Connect: **Code in JavaScript → Insert row**
18. Save and activate (optional) the sub-workflow.

---

### B) Create the Parent Workflow: “Image Generation using Riverflow 2.0”

1. Create a new workflow named **Image Generation using Riverflow 2.0**.
2. Add **Form Trigger** node named **On form submission**:
   - Add fields equivalent to the JSON:
     - Model (dropdown: `pro`, `fast`)
     - Font Urls (text)
     - Font Texts (text)
     - Resolution (dropdown default `1K`, options `1K`, `2K`)
     - Init Images (text)
     - Instruction (textarea, required)
     - Aspect Ratio (dropdown default `auto`, include listed ratios)
     - Transparency (checkbox true)
     - Enhance Prompt (checkbox true)
     - Max Iterations (number default 1)
     - Safety Checker (checkbox default true)
     - Super Resolution Refs (text)
     - Number of Outputs (number default 1)
3. Add **Code** node named **Parse inputs given from form**:
   - Implement:
     - normalization (lowercase + underscores)
     - comma-splitting for array fields
     - boolean checkbox conversion
     - validations (`max_iterations<=3`, `number_of_outputs<=4`, `font_urls<=2`, `init_images<=4 or 10`, `super_resolution_refs<=4`)
     - choose Replicate endpoint based on Model
     - generate `processID` UUID
     - output N duplicated items where N = `number_of_outputs`
4. Connect: **On form submission → Parse inputs given from form**
5. Add **Execute Workflow** node named **Call 'POST + GET requests sub-workflow'**:
   - Select the sub-workflow you created.
   - Mode: **Each**
   - Options: **Wait for sub-workflow = false**
   - Map inputs as:
     - `url = {{$json.url}}`
     - `process_id = {{$json.processID}}`
     - and the other fields as shown in the workflow (including “*_toJsonString” mappings)
6. Connect: **Parse inputs → Call sub-workflow**
7. Add **Code** node **Isolate 1 process id**: return only the first item.
8. Connect: **Call sub-workflow → Isolate 1 process id**
9. Add **Data Table** node **Insert empty process id**:
   - Data Table: **Processes**
   - Insert row with:
     - `process` = the shared process identifier
     - `processcount` = `number_of_outputs`
   - Leave other columns blank.
   - Important: ensure the incoming field names match what you reference (either store `processID` as `process` directly, or reference `processID` correctly).
10. Add **Code** node **Isolate 1 process id1**: return only first item.
11. Connect: **Insert empty process id → Isolate 1 process id1**
12. Add **Wait** node **Wait** with amount `10`.
13. Connect: **Isolate 1 process id1 → Wait**
14. Add **Data Table** node **Get how many processes done**:
   - Operation: **Get**
   - Filter: `process == {{$input.first().json.process}}` (or your corrected field)
15. Connect: **Wait → Get how many processes done**
16. Add **If** node **If number of processes is correct**:
   - Check: `{{$input.all().length}} != {{$input.first().json.processcount + 1}}`
   - True: loop continues
   - False: proceed
17. Connect:
   - **Get how many processes done → If**
   - **If true → Isolate 1 process id1** (loop)
   - **If false → Output Filtering**
18. Add **Code** node **Output Filtering**:
   - Collect `output` strings from all rows
   - Deduplicate
   - Emit items `{url}`
19. Add **HTTP Request** node **Get image/s**:
   - URL: `={{$json.url}}`
20. Connect: **Output Filtering → Get image/s**
21. Save workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add Replicate API key | https://replicate.com/ |
| Data Table is required for parent/sub-workflow communication and completion tracking | Sticky note: “### Data Table … communicate between the sub-workflows and the parent workflow … keep track of if they are complete or not.” |
| Example: “make the text on the shampoo bottle say *Riverflow*” with sample input/output images | Linked images embedded in sticky notes |
| Example: “Create a landing page mockup for my product” with sample input/output images | Linked images embedded in sticky notes |

