Monday.com Useful Utilities

https://n8nworkflows.xyz/workflows/monday-com-useful-utilities-2073


# Monday.com Useful Utilities

### 1. Workflow Overview

This workflow, titled **Monday.com Useful Utilities**, provides a set of modular building blocks designed to extend interaction capabilities with Monday.com beyond what the official Monday.com node in n8n offers. It targets users needing flexible retrieval and manipulation of Monday.com board data, especially for dynamic column access, linked pulse management, subitem handling, and file uploads.

The workflow is organized into multiple logical blocks, each serving a distinct function:

- **1.1 Input Trigger:** Manual trigger to start the workflow.
- **1.2 Item Retrieval:** Fetch a specific board item from Monday.com as the data source.
- **1.3 Column Value Retrieval:**
  - By Column Name
  - By Column ID
- **1.4 Board Relation Handling:** Extract linked pulses from a Board Relation column.
- **1.5 Subitems Handling:** Retrieve and process all subitems of an item.
- **1.6 File Upload to Item:** Upload a binary file to an item's file column (disabled by default).
- **1.7 Data Processing and Splitting:** Code and split nodes to parse JSON values and split arrays into individual items.
- **1.8 Node Merging and Output Preparation:** Merge different data streams for further use.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Trigger

- **Overview:**  
  Initiates the workflow manually for testing or triggering any of the included utilities.

- **Nodes Involved:**  
  - When clicking "Test workflow"

- **Node Details:**  
  - **Name:** When clicking "Test workflow"  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters needed; triggers workflow on manual execution.  
  - **Connections:** Outputs to **GET ITEM** node.  
  - **Edge Cases:** None, as it is a manual trigger.  
  - **Version:** v1

---

#### Block 1.2: Item Retrieval

- **Overview:**  
  Retrieves a specific board item from Monday.com using a fixed item ID. Acts as the base data source for all subsequent column and relation queries.

- **Nodes Involved:**  
  - GET ITEM

- **Node Details:**  
  - **Type:** Monday.com node (boardItem, get operation)  
  - **Configuration:**  
    - `itemId`: Fixed numeric ID "5775061188" (hardcoded for demo, replaceable)  
  - **Credentials:** Monday.com API credentials required.  
  - **Connections:** Outputs to multiple nodes in parallel:  
    - GET BOARD RELATION  
    - PULL SUBITEMS  
    - Convert to File  
    - Merge  
    - COLUMN BY NAME  
    - COLUMN BY ID  
  - **Edge Cases:**  
    - Item not found or invalid ID → node failure.  
    - Authentication errors if credentials are invalid.  
    - Rate limits on Monday.com API.  
  - **Version:** v1

---

#### Block 1.3: Column Value Retrieval

- **Overview:**  
  Provides two utilities to find a column value on an item either by the column's *name* or by its *ID*, avoiding the brittle use of numerical indexes which can change.

- **Nodes Involved:**  
  - COLUMN BY NAME  
  - COLUMN BY ID

- **Node Details:**  

  **COLUMN BY NAME**  
  - Type: Code node (JavaScript)  
  - Function: Searches the item’s `column_values` array for a column matching a specific title ("Zoom Date" by default). Returns the full column object or null.  
  - Key expression: Iterates over `column_values` with `.find(column => column.column.title === columnName)`  
  - Inputs: Receives item JSON from GET ITEM  
  - Outputs: Single JSON containing the matched column object  
  - Edge cases: Column name not found → returns null, which downstream nodes must handle.

  **COLUMN BY ID**  
  - Type: Code node (JavaScript)  
  - Function: Searches the item’s `column_values` array for a column matching a specific ID ("person" by default). Returns the full column object or null.  
  - Key expression: `.find(column => column.id === columnId)`  
  - Inputs: Receives item JSON from GET ITEM  
  - Outputs: Single JSON containing the matched column object  
  - Edge cases: Column ID not found → returns null.

---

#### Block 1.4: Board Relation Handling

- **Overview:**  
  Extracts the pulses linked in a Board Relation column, then looks up full details for each linked pulse.

- **Nodes Involved:**  
  - GET BOARD RELATION  
  - GET LINKEDPULSES  
  - SPLIT LINKED PULSES  
  - PULL LINKEDPULSE

- **Node Details:**  

  **GET BOARD RELATION**  
  - Type: Code node  
  - Function: Finds the column titled "Additional Contacts" in the item’s `column_values` and returns its details (including linked pulse IDs).  
  - Input: Item JSON from GET ITEM  
  - Output: Column JSON or null  
  - Edge cases: Column not found → returns null.

  **GET LINKEDPULSES**  
  - Type: Code node  
  - Function: Parses the `value` field from the previous node’s output to extract `linkedPulseIds` array. Returns the first linkedPulse object.  
  - Input: Output from GET BOARD RELATION  
  - Output: JSON with key `linkedPulse` holding a linked pulse ID.

  **SPLIT LINKED PULSES**  
  - Type: SplitOut node  
  - Function: Splits the `linkedPulse` array into individual items so each can be processed separately.

  **PULL LINKEDPULSE**  
  - Type: Monday.com node (boardItem, get)  
  - Function: Retrieves full details of each linked pulse by item ID.  
  - Input: Receives each linked pulse ID from SPLIT LINKED PULSES.  
  - Credentials: Monday.com API required.  
  - Edge cases: Invalid linked pulse IDs may cause failures.

---

#### Block 1.5: Subitems Handling

- **Overview:**  
  Retrieves the subitems of an item from the "Subitems" column, then fetches each subitem’s details individually.

- **Nodes Involved:**  
  - PULL SUBITEMS  
  - SPLIT SUBITEMS  
  - GET EACH SUBITEM

- **Node Details:**  

  **PULL SUBITEMS**  
  - Type: Code node  
  - Function: Finds the column named "Subitems" and parses its value as JSON to extract subitems array.  
  - Input: Item JSON from GET ITEM  
  - Output: Array of subitems IDs or objects.

  **SPLIT SUBITEMS**  
  - Type: SplitOut node  
  - Function: Splits the subitems array into individual elements for processing.

  **GET EACH SUBITEM**  
  - Type: Monday.com node (boardItem, get)  
  - Function: Gets detailed information for each subitem by its item ID.  
  - Credentials: Monday.com API required.  
  - Input: Subitem IDs from SPLIT SUBITEMS.  
  - Edge cases: Missing subitems or invalid IDs may lead to errors.

---

#### Block 1.6: File Upload to Item

- **Overview:**  
  Uploads a binary file to the files column of a Monday.com item. Disabled by default; meant to be customized and enabled as needed.

- **Nodes Involved:**  
  - Convert to File  
  - Merge  
  - MONDAY UPLOAD (disabled by default)

- **Node Details:**  

  **Convert to File**  
  - Type: ConvertToFile node  
  - Function: Converts JSON data to a file format suitable for upload. Default operation is to convert to JSON file.  
  - Input: Item JSON from GET ITEM  
  - Output: Binary file data

  **Merge**  
  - Type: Merge node  
  - Function: Combines the outputs from **Convert to File** and **GET ITEM** for use in the upload request.  
  - Mode: Combine by position (merging corresponding items).

  **MONDAY UPLOAD**  
  - Type: HTTP Request node  
  - Function: Sends a multipart/form-data POST request to Monday.com's GraphQL file upload endpoint.  
  - Configuration:  
    - URL: `https://api.monday.com/v2/file`  
    - Method: POST  
    - Authentication: Monday.com OAuth2 (different credential than GET ITEM)  
    - Body: GraphQL mutation `add_file_to_column` with variable file upload  
    - Column ID is defaulted to `"file"` but configurable.  
  - Disabled by default; user must provide binary data node and enable.  
  - Edge cases:  
    - Authentication errors.  
    - File size or type restrictions by Monday.com API.  
    - Column ID mismatch or invalid.

---

#### Block 1.7: Data Processing and Splitting

- **Overview:**  
  Contains nodes used to parse JSON strings and split arrays into individual items for iteration.

- **Nodes Involved:**  
  - SPLIT SUBITEMS  
  - SPLIT LINKED PULSES

- **Node Details:**  

  Both are SplitOut nodes used to expand arrays into separate outputs for further processing. Configured to split specific fields containing arrays of IDs or objects.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                         | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                                                          |
|---------------------------|---------------------|---------------------------------------|-------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger      | Start workflow manually                | —                                   | GET ITEM                               |                                                                                                    |
| GET ITEM                  | Monday.com          | Retrieve board item by ID              | When clicking "Test workflow"       | GET BOARD RELATION, PULL SUBITEMS, Convert to File, Merge, COLUMN BY NAME, COLUMN BY ID |                                                                                                    |
| GET BOARD RELATION        | Code                | Extract Board Relation column by name | GET ITEM                           | GET LINKEDPULSES                       |                                                                                                    |
| GET LINKEDPULSES          | Code                | Parse linked pulses from Board Relation | GET BOARD RELATION                | SPLIT LINKED PULSES                    |                                                                                                    |
| SPLIT LINKED PULSES       | Split Out           | Split linked pulses into individual items | GET LINKEDPULSES                 | PULL LINKEDPULSE                       |                                                                                                    |
| PULL LINKEDPULSE          | Monday.com          | Get details of each linked pulse      | SPLIT LINKED PULSES                | —                                      |                                                                                                    |
| PULL SUBITEMS             | Code                | Extract subitems column by name        | GET ITEM                           | SPLIT SUBITEMS                         |                                                                                                    |
| SPLIT SUBITEMS            | Split Out           | Split subitems into individual items  | PULL SUBITEMS                      | GET EACH SUBITEM                       |                                                                                                    |
| GET EACH SUBITEM          | Monday.com          | Get details for each subitem           | SPLIT SUBITEMS                    | —                                      |                                                                                                    |
| Convert to File           | ConvertToFile       | Convert JSON to file for upload        | GET ITEM                          | Merge                                 | Replace with your own binary file output node to upload files (disabled by default).                |
| Merge                     | Merge               | Combine file data and item info for upload | Convert to File, GET ITEM        | MONDAY UPLOAD                         |                                                                                                    |
| MONDAY UPLOAD             | HTTP Request        | Upload file to Monday.com item column  | Merge                             | —                                      | Disabled by default; configure column_id in mutation for custom file column names.                 |
| COLUMN BY NAME            | Code                | Find column value by column name       | GET ITEM                          | —                                      | Edit `columnName` variable inside the node to specify desired column title.                        |
| COLUMN BY ID              | Code                | Find column value by column ID         | GET ITEM                          | —                                      | Edit `columnId` variable inside the node to specify desired column ID.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking \"Test workflow\""  
   - No parameters needed.

2. **Create Monday.com Node to Get Item**  
   - Type: Monday.com (boardItem, get operation)  
   - Name: GET ITEM  
   - Parameters:  
     - `itemId`: Set to your target item ID (e.g., "5775061188")  
   - Credentials: Configure with your Monday.com API credential  
   - Connect output of Manual Trigger to this node.

3. **Create Code Node to Retrieve Column by Name**  
   - Type: Code  
   - Name: COLUMN BY NAME  
   - JavaScript:  
     ```javascript
     const columnName = "Zoom Date"; // Change as needed
     function getColumnValue(item, columnId) {
       const column = item.column_values.find(column => column.column.title === columnId);
       return column || null;
     }
     const columnValue = getColumnValue($input.last().json, columnName);
     return columnValue;
     ```
   - Connect GET ITEM output to this node.

4. **Create Code Node to Retrieve Column by ID**  
   - Type: Code  
   - Name: COLUMN BY ID  
   - JavaScript:  
     ```javascript
     const columnId = "person"; // Change as needed
     function getColumnValue(item, columnId) {
       const column = item.column_values.find(column => column.id === columnId);
       return column || null;
     }
     const columnValue = getColumnValue($input.last().json, columnId);
     return columnValue;
     ```
   - Connect GET ITEM output to this node.

5. **Create Code Node to Get Board Relation Column**  
   - Type: Code  
   - Name: GET BOARD RELATION  
   - JavaScript:  
     ```javascript
     const columnName = "Additional Contacts"; // Change as needed
     function getColumnValue(item, columnId) {
       const column = item.column_values.find(column => column.column.title === columnId);
       return column || null;
     }
     const columnValue = getColumnValue($input.last().json, columnName);
     return columnValue;
     ```
   - Connect GET ITEM output to this node.

6. **Create Code Node to Parse Linked Pulses from Board Relation**  
   - Type: Code  
   - Name: GET LINKEDPULSES  
   - JavaScript:  
     ```javascript
     const data = $input.last().json.value;
     const linkedPulseID = JSON.parse(data).linkedPulseIds;
     return { linkedPulse: linkedPulseID };
     ```
   - Connect GET BOARD RELATION output to this node.

7. **Create SplitOut Node to Split Linked Pulses**  
   - Type: SplitOut  
   - Name: SPLIT LINKED PULSES  
   - Parameters:  
     - Field to Split Out: `linkedPulse`  
   - Connect GET LINKEDPULSES output to this node.

8. **Create Monday.com Node to Get Each Linked Pulse**  
   - Type: Monday.com (boardItem, get)  
   - Name: PULL LINKEDPULSE  
   - Parameters:  
     - `itemId`: Use expression `={{ $json.linkedPulse.linkedPulseId }}`  
   - Credentials: Use your Monday.com account  
   - Connect SPLIT LINKED PULSES output to this node.

9. **Create Code Node to Extract Subitems Column**  
   - Type: Code  
   - Name: PULL SUBITEMS  
   - JavaScript:  
     ```javascript
     const columnName = "Subitems";
     function getColumnValue(item, columnId) {
       const column = item.column_values.find(column => column.column.title === columnId);
       return column || null;
     }
     const columnValue = getColumnValue($input.last().json, columnName);
     return JSON.parse(columnValue.value);
     ```
   - Connect GET ITEM output to this node.

10. **Create SplitOut Node to Split Subitems**  
    - Type: SplitOut  
    - Name: SPLIT SUBITEMS  
    - Field to Split Out: `linkedPulseIds`  
    - Connect PULL SUBITEMS output to this node.

11. **Create Monday.com Node to Get Each Subitem**  
    - Type: Monday.com (boardItem, get)  
    - Name: GET EACH SUBITEM  
    - Parameters:  
      - `itemId`: Expression `={{ $json.linkedPulseIds.linkedPulseId }}`  
    - Credentials: Use your Monday.com account  
    - Connect SPLIT SUBITEMS output to this node.

12. **Create Convert to File Node**  
    - Type: ConvertToFile  
    - Name: Convert to File  
    - Operation: To JSON (default)  
    - Connect GET ITEM output to this node.

13. **Create Merge Node**  
    - Type: Merge  
    - Name: Merge  
    - Mode: Combine by position  
    - Connect Convert to File output to first input  
    - Connect GET ITEM output to second input

14. **Create HTTP Request Node for File Upload** (Optional)  
    - Type: HTTP Request  
    - Name: MONDAY UPLOAD  
    - Disabled by default; enable only if file upload needed  
    - Parameters:  
      - URL: `https://api.monday.com/v2/file`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Authentication: Monday.com OAuth2 credential (different from GET ITEM)  
      - Body Parameters:  
        - `query`: GraphQL mutation with variables for file upload, e.g.,  
          ```
          mutation add_file($file: File!) {
            add_file_to_column(item_id: {{ $input.last().json["id"] }}, column_id:"file", file: $file) {
              id
            }
          }
          ```  
        - `map`: `{"image":"variables.file"}`  
        - `image`: formBinaryData, inputDataFieldName: "data"  
    - Connect Merge output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| To retrieve a column by name, route a Monday.com node that gets an item to **COLUMN BY NAME** and edit the `columnName` in code.    | Workflow setup instructions.                                                                         |
| To retrieve a column by ID, follow Monday.com's instructions to find column IDs, then use **COLUMN BY ID** node with `columnId`.    | https://support.monday.com/hc/en-us/articles/360000225709-Board-item-column-and-automation-or-integration-ID-s |
| To retrieve linked pulses from a Board Relation column, use **GET BOARD RELATION** node, specify the column name, then chain to **PULL LINKEDPULSE** node. | Workflow setup instructions.                                                                         |
| To pull all subitems from an item, use **PULL SUBITEMS** node and connect to **GET EACH SUBITEM** node.                              | Workflow setup instructions.                                                                         |
| To upload a file to an item, replace **Convert to File** node with your binary file source, enable **MONDAY UPLOAD** node, and adjust the column_id if needed. | File upload setup instructions.                                                                     |
| Monday.com API credentials must be created and configured in n8n credential manager before running this workflow.                   | Monday.com account prerequisites.                                                                   |

---

This comprehensive documentation allows advanced users and AI agents to fully understand, reproduce, and customize the Monday.com utilities workflow in n8n.