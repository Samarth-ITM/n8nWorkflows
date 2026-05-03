Store variables between workflow runs using data tables as a key-value store

https://n8nworkflows.xyz/workflows/store-variables-between-workflow-runs-using-data-tables-as-a-key-value-store-13442


# Store variables between workflow runs using data tables as a key-value store

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow implements a **persistent “global variable”** pattern in n8n by using an n8n **Data Table** named **`Globals`** as a **key-value store** with columns `key` and `value`. It retrieves a value by key at the start of each run, falls back to a default when missing, allows your workflow logic to use/update it, then **upserts** (creates or updates) the stored value for the next run.

**Target use cases:**
- Store counters, cursors, timestamps, pagination tokens, last processed IDs, feature flags
- Maintain state between workflow runs without external databases

### Logical Blocks
1. **1.1 Trigger & Variable Retrieval**  
   Starts execution and fetches a stored value from the `Globals` table by key.
2. **1.2 Table-Existence Fallback (Auto-create)**  
   If retrieval fails because the `Globals` table doesn’t exist, create it. If retrieval fails for any other reason, stop with an error.
3. **1.3 Value Decision (Stored vs Default)**  
   If a stored row exists, format it into a usable variable; otherwise set a default.
4. **1.4 Use & Update Variable**
   Placeholder logic node consumes/updates the variable.
5. **1.5 Persist Variable (Upsert)**
   Saves the latest value back to the `Globals` table.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Variable Retrieval

**Overview:**  
Manually triggers the workflow and attempts to read the variable from the `Globals` table using the key `your_variable_name`. Retrieval is configured to continue on error so a fallback path can handle missing tables.

**Nodes involved:**
- **When clicking ‘Execute workflow’**
- **Get Global "your_variable_name"**

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point for manual execution.
- **Configuration choices:** No parameters; standard manual start.
- **Outputs:** Connects to **Get Global "your_variable_name"**.
- **Edge cases:** None (manual execution only).

#### Node: Get Global "your_variable_name"
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — reads a row from a table.
- **Operation:** `get`
- **Table reference:** `Globals` by **name** (easier migration across instances).
- **Filter:** `key` equals `your_variable_name`
- **Limit:** 1
- **Error handling:** `onError: continueErrorOutput` and `alwaysOutputData: true`
  - This is critical: it produces an **error output branch** rather than failing the entire workflow.
- **Outputs / routing:**
  - **Main output (success)** → **Global found**
  - **Error output** → **If not the expected error**
- **Key data used later:**
  - On success, row contains at least `{ key, value }` (depending on table schema and n8n Data Table output format).
  - On error, workflow expects an error message in `{{$json.error}}`.
- **Potential failures / edge cases:**
  - Table doesn’t exist: expected message `"Data table with name \"Globals\" not found"` (handled).
  - Credentials/permissions issue for Data Tables (instance-level feature or RBAC): will go to error branch but **won’t match** expected string → workflow stops.
  - Output schema differences across n8n versions could change where the error message is stored (string match may fail).

**Sticky note applied:**  
“## 1. Get variable”

---

### 2.2 Table-Existence Fallback (Auto-create)

**Overview:**  
If the read failed, the workflow distinguishes between “Globals table missing” (create it) vs any other error (stop).

**Nodes involved:**
- **If not the expected error**
- **Stop and Error**
- **Create Globals table**

#### Node: If not the expected error
- **Type / role:** IF (`n8n-nodes-base.if`) — conditional routing based on error message.
- **Condition logic:**
  - Checks `={{ $json.error }}` **not equals** `Data table with name "Globals" not found`
  - If **true**: error is unexpected → stop
  - If **false**: expected missing-table error → create table
- **Outputs / routing:**
  - **True branch** → **Stop and Error**
  - **False branch** → **Create Globals table**
- **Edge cases / failure types:**
  - If `$json.error` is undefined or formatted differently, the condition may evaluate unexpectedly and route to stop or create incorrectly.
  - The comparison is a strict string match; localization or message changes can break the fallback.

#### Node: Stop and Error
- **Type / role:** Stop and Error (`n8n-nodes-base.stopAndError`) — halts execution with a custom message.
- **Message:** “Unknown error happened while retrieving Globals table”
- **Inputs:** From **If not the expected error** (true branch).
- **Edge cases:** None; always stops.

#### Node: Create Globals table
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — creates the required storage table.
- **Operation:** `create` table
- **Table name:** `Globals`
- **Columns:** `key`, `value` (both implicitly string-like in Data Table column definition)
- **Inputs:** From **If not the expected error** (false branch).
- **Outputs:** No outgoing connection in this workflow JSON (it ends after table creation).
- **Important behavior note:**  
  In the current wiring, after creating the table, the workflow does **not** automatically retry the “get” or proceed to default/usage steps. A second run (or additional connections) is needed if you want seamless continuation.
- **Potential failures / edge cases:**
  - Table already exists (if error branch used incorrectly): create may fail.
  - Permission issues: create fails.

**Sticky note applied:**  
“## Fallback: Create table if it does not exist  
_Needs to run first if previous node failed - hence it is placed above_”

---

### 2.3 Value Decision (Stored vs Default)

**Overview:**  
Determines whether a stored value exists. If yes, maps `$json.value` into a workflow variable field; if not, sets a default value.

**Nodes involved:**
- **Global found**
- **Format value**
- **Set default value**

#### Node: Global found
- **Type / role:** IF (`n8n-nodes-base.if`) — checks whether the retrieved item is not empty.
- **Condition logic:** `={{ $json }}` **object notEmpty** (single value)
- **Outputs / routing:**
  - **True branch** (found) → **Format value**
  - **False branch** (not found) → **Set default value**
- **Inputs:** From **Get Global "your_variable_name"** main output.
- **Edge cases:**
  - Depending on Data Table “get” output, “not found” might be an empty array vs empty object; the check is against `$json` as an object. If Data Table changes its output shape, this condition may misroute.

#### Node: Format value
- **Type / role:** Set (`n8n-nodes-base.set`) — normalizes the retrieved value into a named field for downstream nodes.
- **Assignment:**
  - `your_variable_name` (string) = `{{ $json.value }}`
- **Inputs:** From **Global found** (true branch).
- **Outputs:** → **Do something**
- **Edge cases:**
  - If `value` is missing/null, the result may be `null`/empty string depending on conversion.
  - Type is forced to string; if you store JSON, numbers, etc., you may need parsing/casting.

#### Node: Set default value
- **Type / role:** Set (`n8n-nodes-base.set`) — defines a default when key is missing.
- **Assignment:**
  - `your_variable_name` (string) = `value goes here` (expression is literal string in the template)
- **Inputs:** From **Global found** (false branch).
- **Outputs:** → **Do something**
- **Edge cases:**
  - Same typing caveat: stored values are treated as strings here.

**Sticky note applied:**  
“## 2. Define default value if needed”

---

### 2.4 Use & Update Variable

**Overview:**  
Placeholder node representing your actual business logic. It receives `your_variable_name` and is expected to output an updated `your_variable_name` for persistence.

**Nodes involved:**
- **Do something**

#### Node: Do something
- **Type / role:** No Operation (`n8n-nodes-base.noOp`) — placeholder.
- **Inputs:** From **Format value** and **Set default value**.
- **Outputs:** → **Upsert Global "your_variable_name"**
- **Key expectation:** This node’s output item JSON must contain `your_variable_name`, because the upsert step references it via:
  - `{{ $('Do something').item.json.your_variable_name }}`
- **Edge cases:**
  - If you replace this with multiple items, ensure the upsert mapping matches the intended item.
  - If renamed, update the node reference in the upsert expression.

**Sticky note applied:**  
“## 3. Use variable”

---

### 2.5 Persist Variable (Upsert)

**Overview:**  
Writes the variable back to the `Globals` table using an **upsert** keyed by `key = your_variable_name`, storing the updated `value`.

**Nodes involved:**
- **Upsert Global "your_variable_name"**

#### Node: Upsert Global "your_variable_name"
- **Type / role:** Data Table (`n8n-nodes-base.dataTable`) — persist key/value for next run.
- **Operation:** `upsert`
- **Table reference:** `Globals` by name
- **Filter (match existing row):**
  - `key` equals `your_variable_name`
- **Columns mapping (define below):**
  - `key` = `your_variable_name` (literal)
  - `value` = `{{ $('Do something').item.json.your_variable_name }}`
- **Inputs:** From **Do something**
- **Outputs:** None connected further.
- **Edge cases / failures:**
  - If the `Globals` table doesn’t exist and you didn’t create it first, upsert fails.
  - If `Do something` does not output `your_variable_name`, expression resolves to empty/undefined.
  - Concurrency: simultaneous workflow runs updating same key may overwrite each other (last write wins).

**Sticky note applied:**  
“## 4. Save variable”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get Global "your_variable_name" | ## 1. Get variable |
| Get Global "your_variable_name" | Data Table | Read key from Globals table | When clicking ‘Execute workflow’ | Global found (success), If not the expected error (error) | ## 1. Get variable |
| If not the expected error | IF | Distinguish missing-table vs other errors | Get Global "your_variable_name" (error output) | Stop and Error (true), Create Globals table (false) | ## Fallback: Create table if it does not exist<br>_Needs to run first if previous node failed - hence it is placed above_ |
| Stop and Error | Stop and Error | Halt on unexpected retrieval error | If not the expected error (true) | — | ## Fallback: Create table if it does not exist<br>_Needs to run first if previous node failed - hence it is placed above_ |
| Create Globals table | Data Table | Create Globals key/value table | If not the expected error (false) | — | ## Fallback: Create table if it does not exist<br>_Needs to run first if previous node failed - hence it is placed above_ |
| Global found | IF | Check if value exists | Get Global "your_variable_name" (success) | Format value (true), Set default value (false) | ## 2. Define default value if needed |
| Format value | Set | Map stored `value` to `your_variable_name` | Global found (true) | Do something | ## 2. Define default value if needed |
| Set default value | Set | Create default `your_variable_name` | Global found (false) | Do something | ## 2. Define default value if needed |
| Do something | NoOp | Placeholder for business logic | Format value, Set default value | Upsert Global "your_variable_name" | ## 3. Use variable |
| Upsert Global "your_variable_name" | Data Table | Persist variable to Globals table | Do something | — | ## 4. Save variable |
| Sticky Note1 | Sticky Note | Annotation | — | — | (Annotation) ## 1. Get variable |
| Sticky Note2 | Sticky Note | Annotation | — | — | (Annotation) ## Fallback: Create table if it does not exist<br>_Needs to run first if previous node failed - hence it is placed above_ |
| Sticky Note3 | Sticky Note | Annotation | — | — | (Annotation) ## 2. Define default value if needed |
| Sticky Note7 | Sticky Note | Annotation | — | — | (Annotation) ## 4. Save variable |
| Sticky Note8 | Sticky Note | Annotation | — | — | (Annotation) ## 3. Use variable |
| Sticky Note9 | Sticky Note | Annotation / usage notes | — | — | (Annotation) ## How it works… (full text in Notes section) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **Global Key-Value Store Template** (or your preferred name)
   - Keep it inactive while building.

2. **Add Trigger**
   - Add node: **Manual Trigger**
   - Name it: **When clicking ‘Execute workflow’**

3. **Add Data Table “Get” node**
   - Add node: **Data Table**
   - Name: **Get Global "your_variable_name"**
   - Operation: **Get**
   - Data Table: select by **Name** → `Globals`
   - Filters:
     - Condition: column `key` **equals** `your_variable_name`
     - Match type: **all conditions**
   - Limit: **1**
   - Node settings:
     - Enable **Continue on Fail** (or set “On Error” to continue using error output, depending on UI)
     - Ensure it produces an error output branch (the template uses `continueErrorOutput`)
   - Connect: Manual Trigger → Get Global

4. **Add IF node for retrieval errors**
   - Add node: **IF**
   - Name: **If not the expected error**
   - Condition:
     - Left value: `{{$json.error}}`
     - Operator: **not equals**
     - Right value: `Data table with name "Globals" not found`
   - Connect:
     - **Error output** of “Get Global …” → “If not the expected error”

5. **Add Stop and Error node**
   - Add node: **Stop and Error**
   - Name: **Stop and Error**
   - Error message: `Unknown error happened while retrieving Globals table`
   - Connect:
     - IF **true** output → Stop and Error

6. **Add Data Table “Create table” node**
   - Add node: **Data Table**
   - Name: **Create Globals table**
   - Operation: **Create**
   - Resource: **Table**
   - Table name: `Globals`
   - Columns: add two columns: `key`, `value`
   - Connect:
     - IF **false** output → Create Globals table
   - (Optional but recommended) If you want seamless first-run behavior, connect **Create Globals table** back into the “get/decision” flow (e.g., to “Set default value” or retry “Get Global …”).

7. **Add IF node to check if a row was found**
   - Add node: **IF**
   - Name: **Global found**
   - Condition:
     - Left value: `{{$json}}`
     - Operator: **object not empty**
   - Connect:
     - **Success/main output** of “Get Global …” → Global found

8. **Add Set node to format stored value**
   - Add node: **Set**
   - Name: **Format value**
   - Add field:
     - Name: `your_variable_name`
     - Type: **String**
     - Value: `{{$json.value}}`
   - Connect:
     - Global found **true** → Format value

9. **Add Set node for default value**
   - Add node: **Set**
   - Name: **Set default value**
   - Add field:
     - Name: `your_variable_name`
     - Type: **String**
     - Value: `value goes here`
   - Connect:
     - Global found **false** → Set default value

10. **Add placeholder logic node**
    - Add node: **No Operation**
    - Name: **Do something**
    - Connect:
      - Format value → Do something
      - Set default value → Do something
    - Replace this node later with your real logic, ensuring it outputs `your_variable_name`.

11. **Add Data Table “Upsert” node**
    - Add node: **Data Table**
    - Name: **Upsert Global "your_variable_name"**
    - Operation: **Upsert**
    - Data Table: by **Name** → `Globals`
    - Filter/match condition:
      - `key` equals `your_variable_name`
    - Column mapping:
      - `key` = `your_variable_name`
      - `value` = `{{ $('Do something').item.json.your_variable_name }}`
    - Connect:
      - Do something → Upsert Global

12. **Customize placeholders**
    - Replace every occurrence of `your_variable_name` in:
      - “Get Global …” filter value
      - Set node field name(s)
      - Upsert filter and mapping
    - Replace `value goes here` with your desired default.

13. **No credentials required**
    - n8n Data Tables are internal; no external credentials (like OAuth/API keys) are needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| A data table is used as a key value store having the two columns “key” and “value”. It is referenced by name, which makes it easier to migrate this workflow to other instances. | Sticky note “How it works” |
| 1. At the beginning a variable is retrieved by it’s key; 2. If the variable/key does not exist yet, a default value is set, otherwise the stored value is returned; 3. The variable can now be used and updated; 4. The variable is created/updated in the database for the next workflow run. | Sticky note “How it works” |
| Fallback: If the table doesn’t exist, it automatically gets created. | Sticky note “How it works” |
| Setup: Replace the placeholder **your_variable_name** in the Data Table and Set nodes. | Sticky note “How it works” |
| Customization: Delete the trigger node and place this snippet into your existing workflow. Replace the “Do something” placeholder with your own logic that consumes and later updates the variable; make sure mapping is valid in the save step. Works with multiple variables with slight modification. | Sticky note “How it works” |

