Create a Simple Data Caching System with No External Dependencies

https://n8nworkflows.xyz/workflows/create-a-simple-data-caching-system-with-no-external-dependencies-9369


# Create a Simple Data Caching System with No External Dependencies

---

### 1. Workflow Overview

This workflow implements a simple data caching system using n8n's built-in Data Table feature, requiring no external dependencies. It is designed to provide quick storage and retrieval of JSON-serializable values keyed by a unique string, with an expiration mechanism based on TTL (time-to-live). The workflow is primarily intended to be called as a sub-workflow from other workflows, enabling reusable, centralized caching functionality.

Logical blocks:

- **1.1 Input Reception and Action Determination:** Receives inputs from an external workflow, deciding whether to read from or write to the cache.
- **1.2 Write Cache Entry:** Upserts a cache entry with a key, serialized value, and TTL timestamp.
- **1.3 Read Cache Entry:** Retrieves a cached value by key and checks if it exists and is not expired.
- **1.4 Cache Expiration Handling:** Validates whether cache entries are expired; triggers error if missing or expired.
- **1.5 Scheduled Cache Cleaning:** Periodically deletes expired cache entries to keep table size manageable.
- **1.6 Output Handling:** Returns the correct cached value or error to the calling workflow.
- **1.7 Documentation and Notes:** Sticky notes providing usage instructions and critical setup information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Action Determination

**Overview:**  
This block receives inputs from an external workflow and determines whether the operation requested is a cache write or read based on the `trueToWrite` boolean input.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Check Action Type  
- Action Write Value (NoOp)  
- Read Action (NoOp)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point; triggered by another workflow with specified inputs (`trueToWrite`, `cacheKey`, `writeValue`, `writeTTLms`).  
  - Key Parameters: Expects boolean `trueToWrite` to decide the action type, string `cacheKey` for cache key, any `writeValue` for writing, and TTL in ms `writeTTLms`.  
  - Connections: Outputs to `Check Action Type`.  
  - Failures: Missing inputs or invalid types could cause expression or runtime failures.

- **Check Action Type**  
  - Type: If Node  
  - Role: Branches workflow based on whether `trueToWrite` is true (write) or false/undefined (read).  
  - Expression: Checks if `trueToWrite` coerced to boolean is true.  
  - Inputs: From `When Executed by Another Workflow`.  
  - Outputs: To `Action Write Value` if true, else to `Read Action`.

- **Action Write Value** (NoOp)  
  - Type: No Operation  
  - Role: Placeholder node to continue the write path.  
  - Inputs: From `Check Action Type`.  
  - Outputs: To `Upsert row(s)`.

- **Read Action** (NoOp)  
  - Type: No Operation  
  - Role: Placeholder node to continue the read path.  
  - Inputs: From `Check Action Type`.  
  - Outputs: To `Get row(s)`.

---

#### 1.2 Write Cache Entry

**Overview:**  
This block handles inserting or updating a cache row with the key, serialized value, and computed TTL.

**Nodes Involved:**  
- Upsert row(s)  
- Return Value

**Node Details:**

- **Upsert row(s)**  
  - Type: Data Table Node  
  - Role: Inserts or updates a row in the "cache" table keyed by `cacheKey`.  
  - Configuration:  
    - Operation: Upsert (update if key exists, else insert)  
    - Columns:  
      - `key`: string equal to input `cacheKey`  
      - `value`: stringified JSON of `writeValue`  
      - `ttl`: current time plus `writeTTLms` (defaults to 10,000 ms if not positive)  
    - Filters: Match by `key` column equal to `cacheKey`  
    - Data Table: Points to a table named "cache" (lowercase) with columns `key` (string), `value` (string), `ttl` (datetime).  
  - Inputs: From `Action Write Value` (NoOp)  
  - Outputs: To `Return Value`  
  - Failures: Requires the "cache" table to exist and be accessible; invalid TTL or serialization failures possible.

- **Return Value**  
  - Type: Set Node  
  - Role: Returns the original `writeValue` back to the caller after write completes.  
  - Configuration: Sets output JSON to the `writeValue` from the input trigger node.  
  - Inputs: From `Upsert row(s)`  
  - Outputs: Workflow output node (implicit)  
  - Failures: Expression errors if input data missing.

---

#### 1.3 Read Cache Entry

**Overview:**  
Retrieves a cache row by key from the table.

**Nodes Involved:**  
- Get row(s)  
- If not in Cache  
- If Expired Cache  
- Return Value from Cache  
- No cache found, use error detection to detect this.

**Node Details:**

- **Get row(s)**  
  - Type: Data Table Node  
  - Role: Fetches one or more rows from the "cache" table matching the given `cacheKey`.  
  - Configuration:  
    - Operation: Get  
    - Filters: `key` equals `cacheKey`  
    - Data Table: Same "cache" table as write node.  
  - Inputs: From `Read Action` (NoOp)  
  - Outputs: To `If not in Cache`  
  - Failures: Table missing or inaccessible, no matching key returns empty dataset.

- **If not in Cache**  
  - Type: If Node  
  - Role: Checks if the result from `Get row(s)` is empty (no cache entry found).  
  - Condition: Checks if input JSON object is empty.  
  - Inputs: From `Get row(s)`  
  - Outputs:  
    - If True (empty): To `No cache found, use error detection to detect this.` (error node)  
    - If False (found): To `If Expired Cache`

- **If Expired Cache**  
  - Type: If Node  
  - Role: Checks if the cache entry's `ttl` is after the current time (i.e., still valid).  
  - Condition: Verifies `$now` is after the cache entry's TTL date/time — if current time is after TTL, cache is expired.  
  - Inputs: From `If not in Cache` (False output)  
  - Outputs:  
    - If True (expired): To error node `No cache found, use error detection to detect this.`  
    - If False (valid): To `Return Value from Cache`

- **Return Value from Cache**  
  - Type: Set Node  
  - Role: Parses the stored JSON string from the cache row's `value` column and outputs it as JSON.  
  - Configuration: Uses `JSON.parse()` on the `value` field from the row returned by `Get row(s)`.  
  - Inputs: From `If Expired Cache` (False output)  
  - Outputs: Workflow output node (implicit)  
  - Failures: Parsing errors if cached string corrupted.

- **No cache found, use error detection to detect this.**  
  - Type: Stop and Error Node  
  - Role: Throws an error with message "No Entry or Expired Cache Item." if cache entry is missing or expired.  
  - Inputs: From both `If not in Cache` (True output) and `If Expired Cache` (True output).  
  - Outputs: Terminates workflow with error.  
  - Failures: This node intentionally throws errors to signal missing cache.

---

#### 1.4 Cache Expiration Handling

**Overview:**  
This logic is embedded in the read flow above, validating TTL and forcing error if cache expired or missing.

**Nodes Involved:**  
- If Expired Cache  
- No cache found, use error detection to detect this.

(Note: Covered in block 1.3)

---

#### 1.5 Scheduled Cache Cleaning

**Overview:**  
Periodically deletes all expired rows from the cache table to control data growth and maintain cache hygiene.

**Nodes Involved:**  
- 1 Hour Clean for Cache Table  
- Drop all rows with expired cache entires

**Node Details:**

- **1 Hour Clean for Cache Table**  
  - Type: Schedule Trigger  
  - Role: Triggers the cache cleaning sequence every hour.  
  - Configuration: Interval set to every 1 hour (field: hours).  
  - Outputs: To `Drop all rows with expired cache entires`.

- **Drop all rows with expired cache entires**  
  - Type: Data Table Node  
  - Role: Deletes all cache rows where TTL is less than the current time (expired entries).  
  - Configuration:  
    - Operation: Delete Rows  
    - Filters: `ttl` less than `$now` (expired)  
    - Option: `dryRun` is true, which means no actual deletion occurs (likely for testing; this should be set to false in production).  
    - Data Table: Same "cache" table.  
  - Inputs: From `1 Hour Clean for Cache Table`  
  - Outputs: None (end of cleaning path)  
  - Failures: Table access errors; dry run means no real deletions, so adjust for production.

---

#### 1.6 Output Handling

**Overview:**  
Outputs the cached value or the written value back to the calling workflow, or throws an error if cache miss/expiration occurs.

**Nodes Involved:**  
- Return Value  
- Return Value from Cache  
- No cache found, use error detection to detect this.

(Note: Covered in blocks 1.2 and 1.3)

---

#### 1.7 Documentation and Notes

**Overview:**  
Provides critical instructions, requirements, and usage guidelines for users and developers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - Content:  
    - States requirement for Beta version of n8n Tables.  
    - Specifies the need to create a table named "cache" (all lowercase) with columns:  
      - `key` (string)  
      - `ttl` (datetime)  
      - `value` (string)  
    - Instructs to update the two Data Table nodes to use this table.  
  - Position: Top-left corner to catch attention early.

- **Sticky Note1**  
  - Content:  
    - Provides usage instructions:  
      - How to call this workflow via Execute Sub-Flow with the specified inputs.  
      - Optional activation for hourly cache cleaning.  
      - Outputs: JSON-parsed cached values or errors if missing.  
      - Inputs required for read and write actions, with details on parameters and defaults.  
  - Positioned prominently for user reference.

---

### 3. Summary Table

| Node Name                                | Node Type                   | Functional Role                          | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                  |
|-----------------------------------------|-----------------------------|----------------------------------------|-------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow        | Execute Workflow Trigger     | Entry point receiving inputs            | N/A                           | Check Action Type                      |                                                                                              |
| Check Action Type                        | If                          | Determines read or write operation       | When Executed by Another Workflow | Action Write Value, Read Action       |                                                                                              |
| Action Write Value                       | No Operation                | Placeholder for write path                | Check Action Type              | Upsert row(s)                         |                                                                                              |
| Upsert row(s)                           | Data Table                  | Inserts or updates cache entry            | Action Write Value             | Return Value                          | Requires "cache" table with specific schema (see Sticky Note)                               |
| Return Value                            | Set                         | Returns written value to caller           | Upsert row(s)                 | N/A                                  |                                                                                              |
| Read Action                            | No Operation                | Placeholder for read path                 | Check Action Type              | Get row(s)                           |                                                                                              |
| Get row(s)                             | Data Table                  | Retrieves cache entry by key               | Read Action                   | If not in Cache                      | Requires "cache" table with specific schema (see Sticky Note)                               |
| If not in Cache                        | If                          | Checks if cache entry exists               | Get row(s)                    | No cache found..., If Expired Cache   |                                                                                              |
| No cache found, use error detection... | Stop and Error              | Throws error if cache miss or expired     | If not in Cache, If Expired Cache | N/A                                |                                                                                              |
| If Expired Cache                      | If                          | Checks if cache entry TTL expired          | If not in Cache               | No cache found..., Return Value from Cache |                                                                                              |
| Return Value from Cache               | Set                         | Returns cached JSON-parsed value           | If Expired Cache              | N/A                                  |                                                                                              |
| 1 Hour Clean for Cache Table           | Schedule Trigger            | Triggers hourly cache cleaning             | N/A                           | Drop all rows with expired cache entires |                                                                                              |
| Drop all rows with expired cache entires | Data Table                  | Deletes expired cache entries               | 1 Hour Clean for Cache Table  | N/A                                  | Dry run enabled; disable dryRun for actual deletions                                        |
| Sticky Note                          | Sticky Note                 | Provides setup requirements and instructions | N/A                           | N/A                                  | Requires Beta n8n Tables; create "cache" table with columns: key, ttl, value                  |
| Sticky Note1                         | Sticky Note                 | Provides usage instructions and notes       | N/A                           | N/A                                  | Explains inputs, outputs, and optional hourly cleaning activation                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Cache Table in n8n Beta Tables:**
   - Name: `cache` (all lowercase)
   - Columns:
     - `key` (string)
     - `ttl` (datetime)
     - `value` (string)

2. **Create the Trigger Node:**
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.
   - Configure inputs:
     - `trueToWrite` (boolean)
     - `cacheKey` (string)
     - `writeValue` (any)
     - `writeTTLms` (number)
   - Position it as the workflow entry point.

3. **Add an If Node to Determine Action Type:**
   - Name: `Check Action Type`
   - Condition: Expression checking if `trueToWrite` coerced boolean is true.
   - Connect output from `When Executed by Another Workflow` to this node.

4. **Add Two No Operation Nodes as Placeholders:**
   - `Action Write Value` (for true path)
   - `Read Action` (for false path)
   - Connect `Check Action Type` true output to `Action Write Value`.
   - Connect `Check Action Type` false output to `Read Action`.

5. **Write Path Setup:**
   - Add a **Data Table** node named `Upsert row(s)`.
   - Operation: Upsert
   - Data Table: Select the `cache` table.
   - Columns definition (mapping mode "defineBelow"):
     - `key`: Expression `={{ $json.cacheKey }}`
     - `ttl`: Expression `={{ $now.plus($json.writeTTLms > 0 ? $json.writeTTLms : 10000) }}`
     - `value`: Expression `={{ JSON.stringify($json.writeValue) }}`
   - Filters: Key equals `cacheKey`.
   - Connect `Action Write Value` output to this node.

6. **Return Written Value:**
   - Add a **Set** node named `Return Value`.
   - Mode: Raw
   - JSON Output: Expression `={{ $('When Executed by Another Workflow').item.json.writeValue }}`
   - Connect output of `Upsert row(s)` to this node.

7. **Read Path Setup:**
   - Add a **Data Table** node named `Get row(s)`.
   - Operation: Get
   - Data Table: `cache`
   - Filters: Key equals `cacheKey`
   - Connect `Read Action` output to this node.

8. **Check If Cache Entry Exists:**
   - Add an **If** node named `If not in Cache`.
   - Condition: Check if input JSON is empty (object empty).
   - Connect `Get row(s)` output to `If not in Cache`.

9. **Add an Error Node for Missing Cache:**
   - Add a **Stop and Error** node named `No cache found, use error detection to detect this.`
   - Error Message: "No Entry or Expired Cache Item."
   - Connect `If not in Cache` true output to this node.

10. **Add an If Node to Check Cache Expiration:**
    - Add an **If** node named `If Expired Cache`.
    - Condition: Check if `$now` is after `ttl` field in ISO format, meaning expired.
      - Use expression: `$now > DateTime.fromISO($json.ttl)`
    - Connect `If not in Cache` false output to this node.

11. **Connect Expiration True Output to Error Node:**
    - Connect `If Expired Cache` true output to `No cache found, use error detection to detect this.`

12. **Return Cached Value:**
    - Add a **Set** node named `Return Value from Cache`.
    - Mode: Raw
    - JSON Output: Expression `={{ JSON.parse($('Get row(s)').item.json.value) }}`
    - Connect `If Expired Cache` false output to this node.

13. **Scheduled Cache Cleaning Setup:**
    - Add a **Schedule Trigger** node named `1 Hour Clean for Cache Table`.
    - Set interval to every 1 hour.

14. **Add Data Table Delete Rows Node:**
    - Add a **Data Table** node named `Drop all rows with expired cache entires`.
    - Operation: Delete Rows
    - Data Table: `cache`
    - Filter: `ttl` less than `$now`
    - Ensure `dryRun` is set to false to enable actual deletion in production.
    - Connect `1 Hour Clean for Cache Table` output to this node.

15. **Add Sticky Notes:**
    - Add two **Sticky Note** nodes:
      - One describing the requirement to create the `cache` table with exact schema and Beta Tables requirement.
      - Another providing usage instructions, input parameters, outputs, and optional hourly cleaning activation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires the Beta version of n8n Tables to function correctly.                                                           | Sticky Note content specifying requirement for Beta Tables.                                                                            |
| Cache table must be named `cache` with columns `key` (string), `ttl` (datetime), and `value` (string).                                  | Sticky Note content specifying schema and naming conventions.                                                                           |
| Users must call this workflow via the Execute Sub-Flow node with inputs: `cacheKey`, `trueToWrite` (boolean), `writeValue`, `writeTTLms`.| Sticky Note1 content explaining usage and input parameters.                                                                             |
| Default TTL for cache entries is 10,000 milliseconds (10 seconds) if no or invalid TTL is provided.                                     | Logic in `Upsert row(s)` node setting TTL with fallback.                                                                                |
| The scheduled cache cleaning currently has `dryRun` enabled; disable this to allow actual deletion of expired cache entries.            | Comment in `Drop all rows with expired cache entires` node parameters.                                                                 |

---

**Disclaimer:** The text above is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.