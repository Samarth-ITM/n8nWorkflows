Migrate large Hugging Face datasets to MongoDB with a looping subworkflow

https://n8nworkflows.xyz/workflows/migrate-large-hugging-face-datasets-to-mongodb-with-a-looping-subworkflow-12338


# Migrate large Hugging Face datasets to MongoDB with a looping subworkflow

## 1. Workflow Overview

**Purpose:** Migrate large Hugging Face datasets into MongoDB by fetching rows in **batches** (offset/length pagination) and looping until no more rows are returned. The batch ingestion is done via a **looping main workflow** that repeatedly calls a **subworkflow** responsible for fetching, transforming, and inserting one batch.

**Target use cases**
- Importing large public datasets from Hugging Face Datasets Server into MongoDB without loading everything at once
- Reliable pagination-based ingestion with a stop condition
- Reusable “batch insert” subworkflow pattern

### 1.1 Main workflow: Configuration & loop orchestration
Defines dataset parameters, calls the subworkflow for one batch, checks whether the batch had rows, and repeats with the next offset until completion.

### 1.2 Subworkflow: Fetch → split → transform → insert batch
Fetches a page of rows from Hugging Face, splits the returned array into individual documents, removes Hugging Face `_id` so MongoDB can generate its own ObjectId, inserts into MongoDB, and returns info required by the loop (new offset and `rows_count`).

---

## 2. Block-by-Block Analysis

### Block A — Start & initial configuration (Main workflow)
**Overview:** Starts the process manually and sets the initial pagination and dataset parameters (offset/length/dataset/config/split/collection).  
**Nodes involved:** `Trigger_Manual`, `Config_Start`

#### Node: Trigger_Manual
- **Type / role:** Manual Trigger — entry point for human-triggered execution.
- **Configuration:** No parameters.
- **Outputs:** One empty item to `Config_Start`.
- **Edge cases:** None.

#### Node: Config_Start
- **Type / role:** Set — initializes run configuration.
- **Configuration choices (interpreted):**
  - `offset = 0`
  - `length = 100` (batch size)
  - `dataset = "MongoDB/airbnb_embeddings"`
  - `config = "default"`
  - `split = "train"`
  - `collection_name = "airbnb"`
- **Outputs:** Sends config to `InsertBatch`.
- **Edge cases:**
  - Wrong dataset/config/split causes Hugging Face request failures downstream.
  - Too large `length` can cause Hugging Face API slowdown/timeouts or oversized Mongo inserts.

**Sticky note covering this block**
- **Sticky Note3:** Explains overall operation and setup steps (HF URL, Mongo credentials, subworkflow ID, config).

---

### Block B — Batch execution via subworkflow (Main workflow)
**Overview:** Calls the subworkflow once per loop iteration, waits for it to finish, then decides whether to continue.  
**Nodes involved:** `InsertBatch`, `ContinueLoop?`, `Stop`

#### Node: InsertBatch
- **Type / role:** Execute Workflow — calls a separate workflow (“Hg_subworkflow”) per batch.
- **Configuration choices:**
  - **Mode:** `each` (runs once per incoming item; here there is typically one item representing the current offset/length).
  - **Wait for subworkflow:** enabled (`waitForSubWorkflow: true`) so the loop decision can use subworkflow output.
  - **Workflow selected:** cached name `Hg_subworkflow`, id value `P4SdjAY71rPIh9OB` (must exist in your n8n instance).
  - **Inputs mapped to subworkflow:**
    - `offset, length, dataset, config, split, collection_name` from the current main item.
  - **Type conversion:** `convertFieldsToString: true` (be careful: may coerce numbers to strings depending on behavior/version).
- **Input:** From `Config_Start` initially, then from `ContinueLoop?` on subsequent iterations.
- **Output:** To `ContinueLoop?`.
- **Version-specific considerations:** Node typeVersion `1.3`—UI and input mapping differ across n8n versions; confirm the “workflowInputs” mapping after import.
- **Edge cases / failures:**
  - Missing/incorrect subworkflow ID or permissions → execution fails.
  - If the subworkflow returns unexpected fields (e.g., missing `rows_count`), the loop condition may break.

#### Node: ContinueLoop?
- **Type / role:** IF — determines whether another batch should run.
- **Condition:** `rows_count != 0`
  - Left: `{{$json.rows_count}}`
  - Right: `0`
- **True output:** Goes back to `InsertBatch` (loop continues).
- **False output:** Goes to `Stop`.
- **Edge cases:**
  - If `rows_count` is undefined/null, strict validation may error or evaluate unexpectedly depending on n8n IF behavior/version. Ensure subworkflow always sets `rows_count` (including “no rows” case).

#### Node: Stop
- **Type / role:** NoOp — terminates the workflow path cleanly.
- **Edge cases:** None.

**Sticky note covering this block**
- **Sticky Note5:** “Loop & orchestration … repeat until no rows remain.”

---

### Block C — Subworkflow entry & Hugging Face fetch (Subworkflow)
**Overview:** Receives pagination/config inputs from the main workflow, fetches a batch of rows from Hugging Face Datasets Server using query parameters.  
**Nodes involved:** `SubTrigger`, `HF_FetchRows`, `Extract_Rows`, `HasRows?`

#### Node: SubTrigger
- **Type / role:** Execute Workflow Trigger — entry point for subworkflow calls.
- **Expected inputs (declared):**
  - `offset` (number)
  - `length` (number)
  - `dataset` (string)
  - `config` (string)
  - `split` (string)
  - `collection_name` (string)
- **Output:** Passes these values to `HF_FetchRows`.
- **Edge cases:** If main workflow doesn’t pass required fields, downstream expressions referencing `SubTrigger` will fail.

#### Node: HF_FetchRows
- **Type / role:** HTTP Request — calls Hugging Face Datasets Server endpoint `/rows`.
- **Configuration choices:**
  - **URL:** `https://datasets-server.huggingface.co/rows`
  - **Query params:**
    - `dataset = {{$json.dataset}}`
    - `config = {{$json.config}}`
    - `split = {{$json.split}}`
    - `offset = {{$('SubTrigger').item.json.offset}}`
    - `length = {{$('SubTrigger').item.json.length}}`
  - **Retry on fail:** enabled
  - **On error:** `continueRegularOutput` (workflow continues even if request fails)
- **Input:** From `SubTrigger`
- **Output:** To `Extract_Rows`
- **Edge cases / failures:**
  - API errors (429 rate limit, 5xx, invalid dataset/config/split).
  - Because errors continue on the regular output, downstream nodes may receive an error-shaped payload or missing `rows`, leading to false negatives or expression issues. Consider adding explicit error checks if needed.
  - Large responses may cause memory pressure.

#### Node: Extract_Rows
- **Type / role:** Set — extracts the `rows` array from the HF response.
- **Configuration:** sets `rows = {{$json.rows}}` (as an array).
- **Output:** To `HasRows?`.
- **Edge cases:** If HF response doesn’t contain `rows`, `rows` becomes `undefined`.

#### Node: HasRows?
- **Type / role:** IF — checks whether the fetched batch returned rows.
- **Condition:** `{{$json.rows}}` is **array not empty**.
- **True output:** to `Row_Splitter`
- **False output:** to `NoRows_Offset`
- **Edge cases:** If `rows` is not an array (e.g., error response), strict type validation could fail or treat it as empty.

**Sticky note covering this block**
- **Sticky Note2:** “Fetch & extract … Fetch rows from HF API, extract array, and split into items.”

---

### Block D — Split, transform, insert into MongoDB (Subworkflow)
**Overview:** Splits the rows array into individual items, removes Hugging Face `_id`, and inserts documents into MongoDB. Then aggregates results to compute `rows_count` and the next `offset`.  
**Nodes involved:** `Row_Splitter`, `Transform_RemoveId_AddMeta`, `Mongo_InsertOrUpsert`, `Aggregate`, `setOffset`

#### Node: Row_Splitter
- **Type / role:** Split Out — converts `rows[]` into one item per array element.
- **Configuration:** `fieldToSplitOut = "rows"`
- **Input:** From `HasRows?` (true branch)
- **Output:** To `Transform_RemoveId_AddMeta`
- **Edge cases:** If `rows` is large, item explosion may affect performance; batch size (`length`) controls this.

#### Node: Transform_RemoveId_AddMeta
- **Type / role:** Code — transforms each row into a MongoDB document and removes `_id`.
- **Logic (interpreted):**
  - Reads `item.json.row` (note: Hugging Face `/rows` typically returns elements shaped like `{ row: {...}, ... }`).
  - If `row` is missing/not an object → returns `null` for that item (filtered out).
  - Removes `_id` via destructuring: `const { _id, ...doc } = row;`
  - Emits `{ json: doc }`
- **Input:** Each split row item
- **Output:** To `Mongo_InsertOrUpsert`
- **Edge cases:**
  - If HF response schema changes (no `row` key), all items become null and nothing inserts.
  - Removing `_id` avoids Mongo duplicate key conflicts when re-importing; however, it also prevents deterministic upserts unless you add your own unique key.

#### Node: Mongo_InsertOrUpsert
- **Type / role:** MongoDB — inserts documents.
- **Configuration choices:**
  - **Operation:** `insert` (despite node name implying upsert)
  - **Collection:** hard-coded as `airbnb` (not using `collection_name` input)
  - **Fields:** dynamic list `{{ Object.keys($json).join(',') }}` (all keys in current document)
  - **Dot notation:** disabled (`useDotNotation: false`)
- **Input:** Transformed documents from `Transform_RemoveId_AddMeta`
- **Output:** To `Aggregate`
- **Credentials:** Requires MongoDB connection configured in n8n.
- **Edge cases / failures:**
  - Collection mismatch: the workflow parameter `collection_name` is not used; changing `Config_Start.collection_name` will not affect inserts unless you also change this node.
  - Insert failures due to schema (e.g., invalid key names containing `.` or starting with `$`), document size > 16MB, or auth/connection issues.
  - If you rerun, duplicates may occur (no upsert, `_id` removed).

#### Node: Aggregate
- **Type / role:** Aggregate — consolidates all insert results into a single item.
- **Configuration:** `aggregateAllItemData` (collects data across all incoming items).
- **Input:** From `Mongo_InsertOrUpsert`
- **Output:** To `setOffset`
- **Edge cases:** Large aggregation may be memory heavy if Mongo returns large payloads per insert.

#### Node: setOffset
- **Type / role:** Set — computes loop control fields to return to main workflow.
- **Configuration (key assignments):**
  - `rows_count = {{$items("Row_Splitter").length}}` (number of rows processed in this batch)
  - `offset = {{ $('SubTrigger').item.json.offset + $('SubTrigger').item.json.length }}` (next offset)
  - Pass-through:
    - `length, dataset, config, split, collection_name` from `SubTrigger`
- **Input:** From `Aggregate`
- **Output:** (implicit end of subworkflow; returned to `InsertBatch`)
- **Edge cases:**
  - If `Row_Splitter` didn’t run (e.g., on empty rows), `$items("Row_Splitter")` would be 0 items; however this node is only reached in the “has rows” branch, so it’s OK.
  - Offset calculation assumes the API always returns exactly `length` rows when available. If the last page returns fewer rows, you still advance by `length` (fine, because stop condition uses `rows_count`).

**Sticky note covering this block**
- **Sticky Note4:** “Transform & insert … Remove HF _id … insert documents into MongoDB.”

---

### Block E — Empty-batch handling (Subworkflow)
**Overview:** If Hugging Face returns no rows, explicitly sets `rows_count = 0` so the main workflow stops looping.  
**Nodes involved:** `NoRows_Offset`

#### Node: NoRows_Offset
- **Type / role:** Set — stop signal for the loop.
- **Configuration:** sets `rows_count = 0`
- **Input:** From `HasRows?` (false branch)
- **Output:** (implicit end of subworkflow; returned to `InsertBatch`)
- **Edge cases:** Offset is not returned/updated here; main workflow relies only on `rows_count` to stop.

**Sticky note covering this block**
- **Sticky Note9:** “Set rows_count to zero”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger_Manual | Manual Trigger | Manual entry point | — | Config_Start | ## How it works: 1- This workflow automates the migration of large datasets by fetching data in batches from the Hugging Face API. 2- It begins with a configuration node where you define the dataset, split, and batch length. 3- The main workflow triggers a subworkflow that fetches specific rows, extracts the data, and splits the array into individual items. 4- A Code node transforms the data by removing the Hugging Face _id to allow MongoDB to generate its own unique ObjectIDs. 5- After inserting the batch into MongoDB, the workflow calculates a new offset and loops until all rows have been processed. |
| Config_Start | Set | Initialize pagination + dataset parameters | Trigger_Manual | InsertBatch | ## How it works: (same note as above) |
| InsertBatch | Execute Workflow | Call subworkflow per batch (wait) | Config_Start; ContinueLoop? (true) | ContinueLoop? | Loop & orchestration — Control offset/length, call subworkflow per batch, repeat until no rows remain. |
| ContinueLoop? | IF | Decide whether to continue loop | InsertBatch | InsertBatch (true); Stop (false) | Loop & orchestration — Control offset/length, call subworkflow per batch, repeat until no rows remain. |
| Stop | NoOp | End of main workflow | ContinueLoop? (false) | — | Loop & orchestration — Control offset/length, call subworkflow per batch, repeat until no rows remain. |
| SubTrigger | Execute Workflow Trigger | Subworkflow entry (receives inputs) | — | HF_FetchRows | Fetch & extract — Fetch rows from HF API, extract array, and split into items. |
| HF_FetchRows | HTTP Request | Fetch rows batch from HF datasets-server `/rows` | SubTrigger | Extract_Rows | Fetch & extract — Fetch rows from HF API, extract array, and split into items. |
| Extract_Rows | Set | Extract `rows` array from HF response | HF_FetchRows | HasRows? | Fetch & extract — Fetch rows from HF API, extract array, and split into items. |
| HasRows? | IF | Branch on whether rows were returned | Extract_Rows | Row_Splitter (true); NoRows_Offset (false) | Fetch & extract — Fetch rows from HF API, extract array, and split into items. |
| Row_Splitter | Split Out | Split `rows[]` into individual items | HasRows? (true) | Transform_RemoveId_AddMeta | Transform & insert — Remove HF _id, add metadata, then insert documents into MongoDB. |
| Transform_RemoveId_AddMeta | Code | Remove `_id` and shape docs | Row_Splitter | Mongo_InsertOrUpsert | Transform & insert — Remove HF _id, add metadata, then insert documents into MongoDB. |
| Mongo_InsertOrUpsert | MongoDB | Insert docs into MongoDB collection | Transform_RemoveId_AddMeta | Aggregate | Transform & insert — Remove HF _id, add metadata, then insert documents into MongoDB. |
| Aggregate | Aggregate | Combine results to compute counts/next offset | Mongo_InsertOrUpsert | setOffset | Transform & insert — Remove HF _id, add metadata, then insert documents into MongoDB. |
| setOffset | Set | Compute `rows_count` + next `offset`, pass through config | Aggregate | — (returns to caller) | Transform & insert — Remove HF _id, add metadata, then insert documents into MongoDB. |
| NoRows_Offset | Set | Set `rows_count=0` to stop looping | HasRows? (false) | — (returns to caller) | ### Set rows_count to zero |

---

## 4. Reproducing the Workflow from Scratch

### A) Create the subworkflow (“Hg_subworkflow”)
1. **Create a new workflow** named (for clarity) `Hg_subworkflow`.
2. Add **Execute Workflow Trigger** node named **SubTrigger**:
   - Define inputs: `offset (number)`, `length (number)`, `dataset (string)`, `config (string)`, `split (string)`, `collection_name (string)`.
3. Add **HTTP Request** node named **HF_FetchRows** connected from `SubTrigger`:
   - Method: GET
   - URL: `https://datasets-server.huggingface.co/rows`
   - Enable **Send Query Parameters**
   - Add query params:
     - `dataset` = `{{$json.dataset}}`
     - `config` = `{{$json.config}}`
     - `split` = `{{$json.split}}`
     - `offset` = `{{$('SubTrigger').item.json.offset}}`
     - `length` = `{{$('SubTrigger').item.json.length}}`
   - Turn on **Retry on Fail**
   - Set **On Error** to “Continue (regular output)” if you want the current behavior.
4. Add **Set** node named **Extract_Rows** connected from `HF_FetchRows`:
   - Add field `rows` (Array) with value `{{$json.rows}}`.
5. Add **IF** node named **HasRows?** connected from `Extract_Rows`:
   - Condition: `rows` → “is not empty” (array notEmpty).
6. On **true** branch, add **Split Out** node named **Row_Splitter**:
   - Field to split out: `rows`
7. Add **Code** node named **Transform_RemoveId_AddMeta** connected from `Row_Splitter`:
   - Paste logic that reads `item.json.row`, removes `_id`, returns `{json: doc}` and filters invalid rows.
8. Add **MongoDB** node named **Mongo_InsertOrUpsert** connected from the Code node:
   - Credentials: configure your MongoDB connection (connection string/host, auth, database).
   - Operation: **Insert**
   - Collection: set to your desired collection (current workflow uses `airbnb`; optionally switch to expression `{{$json.collection_name}}` if you want it dynamic).
   - Fields: use expression `{{ Object.keys($json).join(',') }}`
   - Dot notation: off
9. Add **Aggregate** node named **Aggregate** connected from Mongo node:
   - Mode: “Aggregate All Item Data”
10. Add **Set** node named **setOffset** connected from `Aggregate`:
   - `rows_count` = `{{$items("Row_Splitter").length}}`
   - `offset` = `{{$('SubTrigger').item.json.offset + $('SubTrigger').item.json.length}}`
   - Pass through: `length, dataset, config, split, collection_name` from `SubTrigger`.
11. On **false** branch of `HasRows?`, add **Set** node named **NoRows_Offset**:
   - Set `rows_count = 0`
12. **Save** the subworkflow.

### B) Create the main workflow (loop controller)
1. Create a new workflow named: **Migrate large Hugging Face datasets to MongoDB with a looping subworkflow**
2. Add **Manual Trigger** named **Trigger_Manual**.
3. Add **Set** node named **Config_Start** connected from `Trigger_Manual`:
   - `offset=0`, `length=100`, `dataset="MongoDB/airbnb_embeddings"`, `config="default"`, `split="train"`, `collection_name="airbnb"`.
4. Add **Execute Workflow** node named **InsertBatch** connected from `Config_Start`:
   - Select your saved subworkflow (`Hg_subworkflow`)
   - Enable “Wait for subworkflow”
   - Map inputs: `offset, length, dataset, config, split, collection_name` from the current item.
5. Add **IF** node named **ContinueLoop?** connected from `InsertBatch`:
   - Condition: `rows_count` “not equals” `0`
6. Connect **true** output of `ContinueLoop?` back to **InsertBatch** (this forms the loop).
7. Connect **false** output of `ContinueLoop?` to a **NoOp** node named **Stop**.
8. Save the main workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” + setup checklist (HF URL, Mongo credentials, subworkflow ID, config/batch length). | From Sticky Note3 (embedded in the workflow canvas) |
| Fetch & extract: fetch HF rows, extract array, split into items. | From Sticky Note2 |
| Transform & insert: remove HF `_id`, then insert into MongoDB. | From Sticky Note4 |
| Loop & orchestration: call subworkflow per batch, repeat until no rows remain. | From Sticky Note5 |
| Set `rows_count` to zero when no rows are returned. | From Sticky Note9 |