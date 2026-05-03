Retrieve a Monday.com row and all data in a single node

https://n8nworkflows.xyz/workflows/retrieve-a-monday-com-row-and-all-data-in-a-single-node-2086


# Retrieve a Monday.com row and all data in a single node

### 1. Workflow Overview

This workflow, titled **"Retrieve a Monday.com row and all data in a single node"**, is a reusable, advanced building block designed to be called by other n8n workflows. Its main purpose is to fetch detailed data for a specific Monday.com board item (called a "pulse" in Monday.com terminology) given its ID. It returns a comprehensive JSON object containing the item’s name, ID, all column data indexed both by column name and ID, related board item columns, and all associated subitems with their data.

**Target Use Cases:**  
- Workflows requiring detailed Monday.com item data for processing or decision-making.  
- Integration scenarios where a single node output with all related data is preferred over multiple API calls.  
- Automation that needs to traverse relations and subitems of a Monday.com row.

**Logical Blocks:**  
- **1.1 Input Reception:** Receives the pulse ID via JSON input.  
- **1.2 Primary Item Retrieval:** Uses Monday.com API to fetch the main item and its columns.  
- **1.3 Column Data Processing:** Extracts column values indexed by ID and name.  
- **1.4 Relations Extraction and Resolution:** Identifies board relation columns and fetches linked items’ data.  
- **1.5 Subitems Extraction and Resolution:** Extracts subitem IDs from a special column and fetches their data.  
- **1.6 Data Aggregation and Output:** Combines all fetched and processed data into a single JSON structure for output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception  
**Overview:**  
Receives the input JSON containing the `"pulse"` field, which is the Monday.com item ID to retrieve.

**Nodes Involved:**  
- `Execute Workflow Trigger`

**Node Details:**  
- **Type:** Execute Workflow Trigger  
- **Role:** Entry point for the workflow when called from another workflow. It triggers the workflow execution.  
- **Configuration:** No special parameters.  
- **Input/Output:** Outputs incoming JSON containing the `"pulse"` field.  
- **Edge Cases:** Missing or invalid `"pulse"` field in input JSON will cause downstream failures.

---

#### 1.2 Primary Item Retrieval  
**Overview:**  
Fetch the main Monday.com item using the provided pulse ID, retrieve its column values and item metadata.

**Nodes Involved:**  
- `GET ITEM`  
- `GET ALL COLUMNS`  
- `GET ALL COLUMNS2`  
- `PULL SUBITEMS`

**Node Details:**  

- **GET ITEM**  
  - Type: Monday.com API node  
  - Role: Retrieve the full item data by item ID (`pulse` from input)  
  - Configuration: Uses expression `{{ $input.item.json.pulse }}` for itemId parameter  
  - Credentials: Requires Monday.com API credentials  
  - Input: Pulse ID from workflow trigger  
  - Output: Full item JSON including column_values array  
  - Edge Cases: Invalid item ID, API permission errors, rate limiting.

- **GET ALL COLUMNS**  
  - Type: Code node  
  - Role: Converts column_values array into an object indexed by column ID; adds item name and ID  
  - Logic: Iterates over `column_values`, creates object with keys as `item.id`  
  - Output: JSON with `item` (name & id) and `columnValuesById` (columns indexed by ID)  
  - Input: Output from `GET ITEM`  
  - Edge Cases: Missing or empty column_values array.

- **GET ALL COLUMNS2**  
  - Type: Code node  
  - Role: Similar to `GET ALL COLUMNS` but indexes columns by column title (name) and excludes columns of type "subtasks"  
  - Output: JSON with `columnValuesByName` object  
  - Input: Output from `GET ITEM`  
  - Edge Cases: Columns without titles, presence of subtasks column which is specially handled.

- **PULL SUBITEMS**  
  - Type: Code node  
  - Role: Extracts the "Subitems" column's value (JSON string) and parses it to get subitem IDs  
  - Logic: Finds column with title "Subitems" and parses its value as JSON array  
  - Output: Array of subitem IDs  
  - Input: Output from `GET ITEM`  
  - Edge Cases: No "Subitems" column, malformed JSON in column value.

---

#### 1.3 Relations Extraction and Resolution  
**Overview:**  
Extracts all board relation columns from the main item and retrieves data for each linked pulse.

**Nodes Involved:**  
- `GET ALL RELATIONS`  
- `GET LINKEDPULSES1`  
- `SPLIT LINKED PULSES1`  
- `PULL LINKEDPULSE1`  
- `GET ALL COLUMNS3`

**Node Details:**  

- **GET ALL RELATIONS**  
  - Type: Code node  
  - Role: Filters columns to only those where type is `board_relation`  
  - Output: Array of board relation columns  
  - Input: Output from `GET ALL COLUMNS` (which has columns indexed by ID)  
  - Edge Cases: No board_relation columns present.

- **GET LINKEDPULSES1**  
  - Type: Code node (run once per relation item)  
  - Role: Parses relation column's JSON string to extract linked pulse IDs  
  - Output: Object with linkedPulse (array of IDs), id, and name of relation column  
  - Input: Items from `GET ALL RELATIONS`  
  - Edge Cases: Columns with empty or malformed linkedPulseIds.

- **SPLIT LINKED PULSES1**  
  - Type: SplitOut node  
  - Role: Splits the `linkedPulse` array into individual linked pulse IDs for iteration  
  - Output: Multiple items, each with a single linkedPulse ID  
  - Input: Output from `GET LINKEDPULSES1`  
  - Edge Cases: Empty linkedPulse arrays.

- **PULL LINKEDPULSE1**  
  - Type: Monday.com API node  
  - Role: Fetches detailed data for each linked pulse ID  
  - Configuration: Uses expression `{{ $json.linkedPulse.linkedPulseId }}` for `itemId`  
  - Credentials: Monday.com API credentials  
  - Input: Single linkedPulse ID from splitter  
  - Output: Full item JSON for linked pulse  
  - Edge Cases: Invalid linked pulse ID, permissions errors.

- **GET ALL COLUMNS3**  
  - Type: Code node (run once per linked pulse)  
  - Role: Converts fetched linked pulse columns into object indexed by column ID, adds name and id  
  - Output: Object with `item` and `columnValuesById` for linked pulse  
  - Input: Output from `PULL LINKEDPULSE1`  
  - Edge Cases: Same as other code nodes handling columns.

---

#### 1.4 Subitems Extraction and Resolution  
**Overview:**  
Extracts all subitems IDs and fetches detailed data for each subitem, including their column values.

**Nodes Involved:**  
- `SPLIT SUBITEMS1`  
- `GET EACH SUBITEM1`  
- `GET ALL COLUMNS1`  
- `Aggregate1`

**Node Details:**  

- **SPLIT SUBITEMS1**  
  - Type: SplitOut node  
  - Role: Splits the array of subitem IDs into individual items for processing  
  - Input: Output from `PULL SUBITEMS`  
  - Output: Individual subitem IDs  
  - Edge Cases: Empty subitems array.

- **GET EACH SUBITEM1**  
  - Type: Monday.com API node  
  - Role: Fetches full data for each subitem ID  
  - Configuration: Uses expression `{{ $json.linkedPulseIds.linkedPulseId }}` for `itemId`  
  - Credentials: Monday.com API  
  - Edge Cases: Invalid subitem ID, API failures.

- **GET ALL COLUMNS1**  
  - Type: Code node (run once per subitem)  
  - Role: Processes each subitem’s columns into an object indexed by column ID plus name and id  
  - Output: Combined data object for each subitem  
  - Edge Cases: Similar to other column processing nodes.

- **Aggregate1**  
  - Type: Aggregate node  
  - Role: Aggregates all processed subitems into a single JSON array under `subitems` field  
  - Configuration: Uses `aggregateAllItemData` operation  
  - Output: Single JSON with all subitems combined  
  - Edge Cases: No subitems results, partial failures.

---

#### 1.5 Data Aggregation and Final Output  
**Overview:**  
Combines all retrieved data — main item columns, board relations with data, and subitems — into a single JSON output.

**Nodes Involved:**  
- `Merge4`  
- `Aggregate`  
- `Merge2`  
- `Merge`  
- `Merge1`

**Node Details:**  

- **Merge4**  
  - Type: Merge node  
  - Role: Combines board relation columns (from `GET ALL RELATIONS`) with their fetched data (from `GET ALL COLUMNS3`)  
  - Mode: Combine by position  
  - Output: Combined board relation data array

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates combined board relation data into single JSON array under `boardrelations` field  
  - Output: JSON with `boardrelations` field

- **Merge2**  
  - Type: Merge node  
  - Role: Combines main item column data indexed by ID (from `GET ALL COLUMNS`) with column data indexed by name (from `GET ALL COLUMNS2`)  
  - Output: Merged main item column data

- **Merge**  
  - Type: Merge node  
  - Role: Combines aggregated board relations (`Aggregate`) and aggregated subitems (`Aggregate1`)  
  - Output: Combined related data

- **Merge1**  
  - Type: Merge node  
  - Role: Final merge of main item data (`Merge2`) with combined relations and subitems data (`Merge`)  
  - Output: Single comprehensive JSON object with all data fields: item info, columns indexed both ways, board relations with data, subitems with data

- **Sticky Note**  
  - Provides visual explanation that this node combines all data into one JSON output.

---

### 3. Summary Table

| Node Name            | Node Type              | Functional Role                                | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                      |
|----------------------|------------------------|-----------------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger | Execute Workflow Trigger | Entry point, receives input JSON with pulse ID | -                             | GET ITEM                      |                                                                                                |
| GET ITEM             | Monday.com API          | Fetch main item by pulse ID                    | Execute Workflow Trigger       | GET ALL COLUMNS, GET ALL COLUMNS2, PULL SUBITEMS |                                                                                                |
| GET ALL COLUMNS      | Code                   | Index columns by ID, add item name and ID     | GET ITEM                      | GET ALL RELATIONS, Merge2      | "PULL ALL COLUMN DATA AND INDEX BY ID AND NAME"                                                |
| GET ALL COLUMNS2     | Code                   | Index columns by name, exclude subtasks       | GET ITEM                      | Merge2                        |                                                                                                |
| PULL SUBITEMS        | Code                   | Extract subitems column and parse subitem IDs | GET ITEM                      | SPLIT SUBITEMS1               | "PULL ALL SUBITEMS AND THEIR COLUMN DATA"                                                     |
| GET ALL RELATIONS    | Code                   | Extract board relation columns from columns   | GET ALL COLUMNS               | GET LINKEDPULSES1, Merge4      | "PULL ALL BOARDRELATION COLUMNS AND THEIR DATA"                                               |
| GET LINKEDPULSES1    | Code                   | Extract linked pulse IDs from relation columns | GET ALL RELATIONS             | SPLIT LINKED PULSES1          |                                                                                                |
| SPLIT LINKED PULSES1 | SplitOut                | Split linked pulses array to individual IDs   | GET LINKEDPULSES1             | PULL LINKEDPULSE1             |                                                                                                |
| PULL LINKEDPULSE1    | Monday.com API          | Fetch linked pulse data by ID                  | SPLIT LINKED PULSES1          | GET ALL COLUMNS3              |                                                                                                |
| GET ALL COLUMNS3     | Code                   | Index linked pulse columns by ID, add name/id | PULL LINKEDPULSE1             | Merge4                       |                                                                                                |
| Merge4               | Merge                  | Combine relation columns with their fetched data | GET ALL RELATIONS, GET ALL COLUMNS3 | Aggregate                 |                                                                                                |
| Aggregate            | Aggregate               | Aggregate board relation data into one array  | Merge4                       | Merge                       |                                                                                                |
| SPLIT SUBITEMS1      | SplitOut                | Split subitems array to individual IDs        | PULL SUBITEMS                | GET EACH SUBITEM1             |                                                                                                |
| GET EACH SUBITEM1    | Monday.com API          | Fetch subitem data by ID                        | SPLIT SUBITEMS1              | GET ALL COLUMNS1              |                                                                                                |
| GET ALL COLUMNS1     | Code                   | Index subitem columns by ID, add name and ID  | GET EACH SUBITEM1            | Aggregate1                   |                                                                                                |
| Aggregate1           | Aggregate               | Aggregate all subitems into one array          | GET ALL COLUMNS1             | Merge                       |                                                                                                |
| Merge2               | Merge                  | Merge main item columns indexed by ID and by name | GET ALL COLUMNS, GET ALL COLUMNS2 | Merge1                     |                                                                                                |
| Merge                 | Merge                  | Combine aggregated relations and subitems data | Aggregate, Aggregate1        | Merge1                      |                                                                                                |
| Merge1                | Merge                  | Final merging of main item data with relations and subitems | Merge2, Merge              | -                           | "COMBINE ALL DATA INTO ONE JSON OUTPUT"                                                       |
| Edit Fields           | Set                    | Define pulse field to specify which item to retrieve | -                           | Execute Workflow             | "HOW TO USE - Copy nodes, set workflow ID, define pulse in Edit Fields, then Execute Workflow." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Execute Workflow Trigger node:**  
   - No special configuration needed. This is the entry point that waits for external execution.

2. **Create a Monday.com API node named "GET ITEM":**  
   - Operation: Get an item  
   - Resource: boardItem  
   - Item ID: Expression `{{ $input.item.json.pulse }}`  
   - Set Monday.com API credentials.

3. **Create a Code node named "GET ALL COLUMNS":**  
   - Purpose: Create an object with column values indexed by column ID, add item name and id.  
   - Code snippet:  
     ```js
     function createColumnValuesArray(data) {
       const result = {};
       data.forEach(item => {
         const name = item.id;
         result[name] = item;
       });
       return result;
     }
     columns = $input.last().json.column_values;
     data1 = { "name": $input.last().json.name, "id": $input.last().json.id };
     data2 = createColumnValuesArray(columns);
     return { "item": data1, columnValuesById: data2 };
     ```
   - Connect input from "GET ITEM".

4. **Create a Code node named "GET ALL COLUMNS2":**  
   - Purpose: Create column values indexed by column title (name), excluding "subtasks" type.  
   - Code snippet:  
     ```js
     function createColumnValuesArray(data) {
       const result = {};
       data.forEach(item => {
         if (item.type != "subtasks") {
           const name = item.column.title;
           result[name] = item;
         }
       });
       return result;
     }
     columns = $input.last().json.column_values;
     data = createColumnValuesArray(columns);
     return { "columnValuesByName": data };
     ```
   - Connect input from "GET ITEM".

5. **Create a Code node named "PULL SUBITEMS":**  
   - Purpose: Extract and parse the Subitems column to get subitem IDs array.  
   - Code snippet:  
     ```js
     const columnName = "Subitems";
     function getColumnValue(item, columnId) {
       const column = item.column_values.find(column => column.column.title === columnId);
       return column ? column : null;
     }
     const columnValue = getColumnValue($input.last().json, columnName);
     return JSON.parse(columnValue.value);
     ```
   - Connect input from "GET ITEM".

6. **Create a SplitOut node named "SPLIT SUBITEMS1":**  
   - Field to split out: `linkedPulseIds` (the parsed subitems array)  
   - Connect input from "PULL SUBITEMS".

7. **Create a Monday.com API node named "GET EACH SUBITEM1":**  
   - Operation: Get an item  
   - Resource: boardItem  
   - Item ID: Expression `{{ $json.linkedPulseIds.linkedPulseId }}`  
   - Monday.com API credentials required.  
   - Connect input from "SPLIT SUBITEMS1".

8. **Create a Code node named "GET ALL COLUMNS1":**  
   - Purpose: For each subitem, index columns by column ID and add name/id.  
   - Code snippet:  
     ```js
     function createColumnValuesArray(data) {
       const result = {};
       data.forEach(item => {
         const name = item.id;
         result[name] = item;
       });
       return result;
     }
     columns = $input.item.json.column_values;
     data1 = { "name": $input.item.json.name, "id": $input.item.json.id };
     data2 = createColumnValuesArray(columns);
     return { ...data1, ...data2 };
     ```
   - Connect input from "GET EACH SUBITEM1".

9. **Create an Aggregate node named "Aggregate1":**  
   - Operation: `aggregateAllItemData`  
   - Destination field name: `subitems`  
   - Connect input from "GET ALL COLUMNS1".

10. **Create a Code node named "GET ALL RELATIONS":**  
    - Purpose: Extract all columns of type `board_relation`.  
    - Code snippet:  
      ```js
      var data = $input.last().json.columnValuesById;
      var relations = [];
      for (var key in data) {
        if (data[key].type == "board_relation") {
          relations.push(data[key]);
        }
      }
      return relations;
      ```
    - Connect input from "GET ALL COLUMNS".

11. **Create a Code node named "GET LINKEDPULSES1":**  
    - Purpose: Parse each relation column’s linkedPulseIds JSON array.  
    - Mode: Run once per item  
    - Code snippet:  
      ```js
      data = $input.item.json.value;
      id = $input.item.json.id;
      name = $input.item.json.column.title;
      const linkedPulseID = JSON.parse(data).linkedPulseIds;
      return { "linkedPulse": linkedPulseID, "id": id, "name": name };
      ```
    - Connect input from "GET ALL RELATIONS".

12. **Create a SplitOut node named "SPLIT LINKED PULSES1":**  
    - Field to split out: `linkedPulse`  
    - Connect input from "GET LINKEDPULSES1".

13. **Create a Monday.com API node named "PULL LINKEDPULSE1":**  
    - Operation: Get an item  
    - Resource: boardItem  
    - Item ID: Expression `{{ $json.linkedPulse.linkedPulseId }}`  
    - Monday.com API credentials required.  
    - Connect input from "SPLIT LINKED PULSES1".

14. **Create a Code node named "GET ALL COLUMNS3":**  
    - Purpose: For each linked pulse, index columns by column ID and add name/id.  
    - Mode: Run once per item  
    - Code snippet same as "GET ALL COLUMNS" node.  
    - Connect input from "PULL LINKEDPULSE1".

15. **Create a Merge node named "Merge4":**  
    - Mode: Combine by position  
    - Inputs: `GET ALL RELATIONS` and `GET ALL COLUMNS3`  
    - Connect outputs accordingly.

16. **Create an Aggregate node named "Aggregate":**  
    - Operation: `aggregateAllItemData`  
    - Destination field name: `boardrelations`  
    - Connect input from "Merge4".

17. **Create a Merge node named "Merge2":**  
    - Mode: Combine by position  
    - Inputs: `GET ALL COLUMNS` and `GET ALL COLUMNS2`  
    - Connect inputs accordingly.

18. **Create a Merge node named "Merge":**  
    - Mode: Combine by position  
    - Inputs: `Aggregate` (boardrelations) and `Aggregate1` (subitems)  
    - Connect inputs accordingly.

19. **Create a Merge node named "Merge1":**  
    - Mode: Combine by position  
    - Inputs: `Merge2` (main item columns) and `Merge` (relations + subitems)  
    - This node outputs the final combined JSON with all data.

20. **(Optional) Add Sticky Notes:**  
    - Add sticky notes to explain logical blocks for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Prerequisites: Monday.com account and credential, pulse ID of the item to retrieve.                                                                  | Workflow description.                                                                                            |
| To use this workflow from another, use an "Edit Fields" node to set the "pulse" field, then an "Execute Workflow" node with this workflow's ID.      | Workflow setup instructions.                                                                                      |
| The workflow outputs a JSON object combining item info, columns indexed by ID and name, board relations with data, and subitems with their data.     | Workflow overview and purpose.                                                                                   |
| Example JSON output screenshot is included in the original documentation (not reproducible here).                                                     | Original workflow description.                                                                                   |
| Monday.com API credentials must be configured in all Monday.com nodes.                                                                                | Credentials requirement.                                                                                          |
| The approach avoids multiple calls by aggregating relations and subitems data into single output, useful for complex Monday.com integrations.        | General workflow design rationale.                                                                                |
| For troubleshooting, check for valid pulse IDs, correct API credentials, and proper JSON parsing in code nodes.                                      | Error handling recommendations.                                                                                   |
| Useful links for Monday.com API documentation: https://monday.com/developers/v2/                                                                      | External resource for further enhancements or debugging.                                                        |

---

This document fully covers the workflow structure, node-by-node analysis, reproduction instructions, and contextual notes for an advanced Monday.com data retrieval automation in n8n.