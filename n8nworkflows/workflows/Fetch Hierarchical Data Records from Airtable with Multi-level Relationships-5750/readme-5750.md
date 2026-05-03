Fetch Hierarchical Data Records from Airtable with Multi-level Relationships

https://n8nworkflows.xyz/workflows/fetch-hierarchical-data-records-from-airtable-with-multi-level-relationships-5750


# Fetch Hierarchical Data Records from Airtable with Multi-level Relationships

### 1. Workflow Overview

This n8n workflow is designed to **fetch hierarchical records from Airtable**, supporting up to three nested levels of linked records. It retrieves a target record from a specified Airtable base and table, then recursively fetches linked child records (Level 2), and optionally, linked grandchildren records (Level 3) based on user input. The workflow also processes rich text fields converting pseudo-markdown to HTML if requested, and finally returns a comprehensive nested JSON object representing the entire record hierarchy.

Key logical blocks:

- **1.1 Input Reception and Initialization**: Accepts input parameters defining the base, table, record, and which Level 3 links to fetch.
- **1.2 Base Record Retrieval and Schema Analysis**: Fetches the target record and the Airtable schema to understand relationships.
- **1.3 Level 2 Linked Records Fetching**: Identifies and retrieves Level 2 linked records, filtering out unnecessary fields.
- **1.4 Level 3 Linked Records Conditional Fetching**: If configured, iterates over Level 2 records to fetch Level 3 linked records selectively.
- **1.5 Data Aggregation and Assembly**: Aggregates all fetched data into a hierarchical JSON structure.
- **1.6 Rich Text Fields Conversion**: Optionally converts Airtable rich text fields from pseudo-markdown to HTML.
- **1.7 Output Preparation**: Merges all fetched data, including comments, into final output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:** Receives initial workflow inputs containing Airtable base, table, record IDs, and configuration for Level 3 linked fields and HTML conversion.
- **Nodes Involved:** `Execute Workflow Trigger`, `Inputs`
- **Node Details:**

  - **Execute Workflow Trigger**
    - Type: Execute Workflow Trigger (n8n special node)
    - Role: Entry point for triggering this workflow externally.
    - Configuration: No specific parameters.
    - Inputs: None (trigger)
    - Outputs: Connected to `Inputs`
    - Edge Cases: Trigger payload must be valid JSON with required parameters.

  - **Inputs**
    - Type: Set
    - Role: Extracts and sets key input parameters (`base_id`, `table_id`, `record_id`, and `level_3`) from the trigger payload.
    - Configuration: Uses expressions to assign parameters from incoming JSON.
    - Inputs: From `Execute Workflow Trigger`
    - Outputs: To `Airtable - Get Target Record` and `Get Airtable Comments`
    - Edge Cases: Missing or malformed input fields may cause downstream errors.

---

#### 1.2 Base Record Retrieval and Schema Analysis

- **Overview:** Fetch the target Airtable record and retrieve the Airtable base schema to understand table and field relationships.
- **Nodes Involved:** `Airtable - Get Target Record`, `Get Airtable Comments`, `Rename comments`, `Airtable Schema`, `Filter Main Table`, `Split Out Fields`, `Filter Link To fields`, `Add Inverse LinkTo Field name`
- **Node Details:**

  - **Airtable - Get Target Record**
    - Type: Airtable
    - Role: Retrieves the target record specified by input IDs.
    - Configuration: Uses input `base_id`, `table_id`, and `record_id`.
    - Credentials: Airtable API token.
    - Inputs: From `Inputs`
    - Outputs: To `Airtable Schema`

  - **Get Airtable Comments**
    - Type: HTTP Request
    - Role: Fetches comments related to the target record via Airtable API.
    - Configuration: GET request to `/comments` endpoint for the record.
    - Inputs: From `Inputs`
    - Outputs: To `Rename comments`

  - **Rename comments**
    - Type: Set
    - Role: Renames fetched comments to a field `AirtableComments`.
    - Inputs: From `Get Airtable Comments`
    - Outputs: Later merged with main data.
  
  - **Airtable Schema**
    - Type: Airtable
    - Role: Retrieves the schema of the base to map tables and fields.
    - Inputs: From `Airtable - Get Target Record`
    - Outputs: To `Filter Main Table`

  - **Filter Main Table**
    - Type: Filter
    - Role: Filters schema to focus on the main table of interest.
    - Filter Condition: Schema entry with ID matching input `table_id`.
    - Inputs: From `Airtable Schema`
    - Outputs: To `Split Out Fields`

  - **Split Out Fields**
    - Type: Split Out
    - Role: Splits schema fields array into individual items.
    - Inputs: From `Filter Main Table`
    - Outputs: To `Filter Link To fields`

  - **Filter Link To fields**
    - Type: Filter
    - Role: Filters fields of type `multipleRecordLinks` (Airtable link fields).
    - Inputs: From `Split Out Fields`
    - Outputs: To `Add Inverse LinkTo Field name`

  - **Add Inverse LinkTo Field name**
    - Type: Set
    - Role: Adds the inverse link field name from the schema, used for two-way linked records.
    - Uses expression to find inverse field name based on linkedTableId and inverseLinkFieldId in field options.
    - Inputs: From `Filter Link To fields`
    - Outputs: To `Get Linked records without Link To fields Except for Level 3`

- **Edge Cases:**  
  - Schema retrieval errors or changes in Airtable schema may break expressions.  
  - Missing inverse link field definition may cause incomplete fetching of linked records.

---

#### 1.3 Level 2 Linked Records Fetching

- **Overview:** Fetch Level 2 linked records, excluding link fields except those specified for Level 3 fetching.
- **Nodes Involved:** `Get Linked records without Link To fields Except for Level 3`, `If Has Level 3`, `Loop Over L2 records arrays having L3 children to fetch`, `Merge1`, `Repeat L3 links of current L2 table`, `Level 2 records`, `Merge`
- **Node Details:**

  - **Get Linked records without Link To fields Except for Level 3**
    - Type: HTTP Request
    - Role: POST request to Airtable API to list records from linked tables, filtering by linked record IDs.
    - Filters out `multipleRecordLinks` fields except those in `level_3` input.
    - Inputs: From `Add Inverse LinkTo Field name`
    - Outputs: To `If Has Level 3`

  - **If Has Level 3**
    - Type: If
    - Role: Checks if Level 3 linked fields exist and if there are linked records to fetch.
    - Conditions:
      - Checks if any Level 3 fields provided in input exist in the schema of the linked table.
      - Checks if linked records are non-empty.
    - True branch: To `Loop Over L2 records arrays having L3 children to fetch`
    - False branch: To `Merge1`

  - **Loop Over L2 records arrays having L3 children to fetch**
    - Type: Split In Batches
    - Role: Iterates over arrays of Level 2 records that have Level 3 children.
    - Inputs: From `If Has Level 3`
    - Outputs: To `Merge1`, `Repeat L3 links of current L2 table`, and `Level 2 records`

  - **Repeat L3 links of current L2 table**
    - Type: Set
    - Role: Filters the Level 3 links to only those relevant for current Level 2 table.
    - Inputs: From `Loop Over L2 records arrays having L3 children to fetch`
    - Outputs: To `Merge`

  - **Level 2 records**
    - Type: Split Out
    - Role: Splits Level 2 records array into individual records.
    - Inputs: From `Loop Over L2 records arrays having L3 children to fetch`
    - Outputs: To `Merge`

  - **Merge**
    - Type: Merge
    - Role: Combines Level 2 record splits and corresponding Level 3 links.
    - Inputs: From `Level 2 records` and `Repeat L3 links of current L2 table`
    - Outputs: To `Loop Over L2 Records for a given L1 field`

- **Edge Cases:**  
  - API rate limits or paging issues when fetching many linked records.  
  - Empty or missing Level 3 links may skip nested fetch.  
  - Mismatched or changed schema may cause incorrect filtering.

---

#### 1.4 Level 3 Linked Records Conditional Fetching

- **Overview:** For each Level 2 record’s relevant linked fields, fetch Level 3 linked records excluding reverse link fields.
- **Nodes Involved:** `Loop Over L2 Records for a given L1 field`, `Aggregate to record list`, `Split Out L3 Potential Links`, `Prepare HTTP Call`, `Get L3 Linked records without reverse Link To fields`, `Preparing aggregation`, `Aggregate1`, `Code to single field property`, `Merge with original field property`
- **Node Details:**

  - **Loop Over L2 Records for a given L1 field**
    - Type: Split In Batches
    - Role: Iterates over each Level 2 record related to a specific Level 1 field.
    - Inputs: From `Merge`
    - Outputs: To `Aggregate to record list` and `Split Out L3 Potential Links`

  - **Aggregate to record list**
    - Type: Aggregate
    - Role: Aggregates Level 2 records back into a list.
    - Inputs: From `Loop Over L2 Records for a given L1 field`
    - Outputs: To `Loop Over L2 records arrays having L3 children to fetch`

  - **Split Out L3 Potential Links**
    - Type: Split Out
    - Role: Splits Level 3 links array for processing.
    - Inputs: From `Loop Over L2 Records for a given L1 field`
    - Outputs: To `Prepare HTTP Call`

  - **Prepare HTTP Call**
    - Type: Set
    - Role: Prepares parameters for the HTTP request to fetch Level 3 records, including table and field IDs for both Level 2 and Level 3.
    - Inputs: From `Split Out L3 Potential Links`
    - Outputs: To `Get L3 Linked records without reverse Link To fields`

  - **Get L3 Linked records without reverse Link To fields**
    - Type: HTTP Request
    - Role: Fetches Level 3 records excluding reverse link fields.
    - Inputs: From `Prepare HTTP Call`
    - Outputs: To `Preparing aggregation`

  - **Preparing aggregation**
    - Type: Set
    - Role: Prepares data for aggregation by extracting Level 2 record fields, Level 3 records, and Level 2 field names.
    - Inputs: From `Get L3 Linked records without reverse Link To fields`
    - Outputs: To `Aggregate1`

  - **Aggregate1**
    - Type: Aggregate
    - Role: Aggregates Level 2 field names and Level 3 records for final assembly.
    - Inputs: From `Preparing aggregation`
    - Outputs: To `Code to single field property`

  - **Code to single field property**
    - Type: Code (JavaScript)
    - Role: Transforms aggregated data into a single JSON object with Level 3 records assigned to appropriate Level 2 fields.
    - Inputs: From `Aggregate1`
    - Outputs: To `Merge with original field property`

  - **Merge with original field property**
    - Type: Set
    - Role: Combines newly prepared fields with original Level 2 record fields.
    - Inputs: From `Code to single field property`
    - Outputs: To `Loop Over L2 Records for a given L1 field` (loop continues)

- **Edge Cases:**  
  - Large numbers of Level 3 records may cause batching or rate limit issues.  
  - Missing Level 3 field definitions or reverse link IDs may cause incomplete data.  
  - JavaScript code might fail if data structure changes.

---

#### 1.5 Data Aggregation and Assembly

- **Overview:** Combines all fetched linked records into the original target record, building a hierarchical structure.
- **Nodes Involved:** `Aggregate to record list`, `Insert linked records to originRecord & Clean Field Name`, `If markdown to html`, `Rich Text Fields list`, `Markdown to Html`, `Merge2`
- **Node Details:**

  - **Insert linked records to originRecord & Clean Field Name**
    - Type: Code (JavaScript)
    - Role: Inserts all linked Level 2 and Level 3 records into the original target record under correct field names.
    - Inputs: From `Add origin LinkTo`
    - Outputs: To `If markdown to html`

  - **If markdown to html**
    - Type: If
    - Role: Checks if input parameter `to_html` is true to decide if markdown conversion is needed.
    - Inputs: From `Insert linked records to originRecord & Clean Field Name`
    - Outputs: True → `Rich Text Fields list`, False → `Merge2`

  - **Rich Text Fields list**
    - Type: Set
    - Role: Builds an array of all rich text field names across the schema for conversion.
    - Inputs: From `If markdown to html`
    - Outputs: To `Markdown to Html`

  - **Markdown to Html**
    - Type: Code (JavaScript)
    - Role: Recursively converts rich text fields from pseudo-markdown to HTML using the `marked` library.
    - Inputs: From `Rich Text Fields list`
    - Outputs: To `Merge2`

  - **Merge2**
    - Type: Merge
    - Role: Combines main record data (with linked records and optionally converted HTML) and comments.
    - Inputs: From `Rename comments` and either `Markdown to Html` or `Insert linked records to originRecord & Clean Field Name`
    - Outputs: Final output

- **Edge Cases:**  
  - Conversion requires `marked` npm package; missing package causes failure.  
  - Rich text fields absent or empty do not cause errors but result in no conversion.  
  - Large or deeply nested records may impact performance.

---

#### 1.6 Output Preparation

- **Overview:** Final node merges comments and assembled record data, preparing the output for downstream consumption.
- **Nodes Involved:** `Merge2`
- **Node Details:**

  - **Merge2**
    - Type: Merge
    - Role: Combines JSON from record data and comments into a single output.
    - Inputs: From the rich text processing path or directly from record insertion and from comments renaming.
    - Outputs: Final output of the workflow.

- **Edge Cases:**  
  - Missing comments do not block output but yield empty comments array.

---

### 3. Summary Table

| Node Name                                    | Node Type                   | Functional Role                                   | Input Node(s)                              | Output Node(s)                                   | Sticky Note                                                                                              |
|----------------------------------------------|-----------------------------|-------------------------------------------------|--------------------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger                      | Execute Workflow Trigger     | Workflow entry trigger                           | -                                          | Inputs                                          |                                                                                                         |
| Inputs                                       | Set                         | Receives and sets input parameters               | Execute Workflow Trigger                    | Airtable - Get Target Record, Get Airtable Comments | ## Input  
Sample input JSON provided                                                                           |
| Airtable - Get Target Record                  | Airtable                    | Fetches the target record                         | Inputs                                      | Airtable Schema                                  |                                                                                                         |
| Get Airtable Comments                         | HTTP Request                | Retrieves comments for the target record          | Inputs                                      | Rename comments                                 |                                                                                                         |
| Rename comments                              | Set                         | Renames comments field                            | Get Airtable Comments                        | Merge2                                          |                                                                                                         |
| Airtable Schema                              | Airtable                    | Retrieves base schema                             | Airtable - Get Target Record                 | Filter Main Table                               |                                                                                                         |
| Filter Main Table                            | Filter                      | Filters schema for main table                     | Airtable Schema                             | Split Out Fields                                |                                                                                                         |
| Split Out Fields                            | Split Out                   | Splits fields array                               | Filter Main Table                           | Filter Link To fields                           |                                                                                                         |
| Filter Link To fields                       | Filter                      | Filters for linked record fields                  | Split Out Fields                           | Add Inverse LinkTo Field name                   |                                                                                                         |
| Add Inverse LinkTo Field name               | Set                         | Adds inverse link field name                       | Filter Link To fields                       | Get Linked records without Link To fields Except for Level 3 |                                                                                                         |
| Get Linked records without Link To fields Except for Level 3 | HTTP Request                | Fetches Level 2 linked records                     | Add Inverse LinkTo Field name                | If Has Level 3                                  |                                                                                                         |
| If Has Level 3                              | If                          | Checks if Level 3 linked records should be fetched | Get Linked records without Link To fields Except for Level 3 | Loop Over L2 records arrays having L3 children to fetch, Merge1 |                                                                                                         |
| Loop Over L2 records arrays having L3 children to fetch | Split In Batches            | Iterates over L2 records arrays with L3 children | If Has Level 3                              | Merge1, Repeat L3 links of current L2 table, Level 2 records | ## Level 3 children  
Iteration on each L2 record array to fetch Level 3 children                                             |
| Merge1                                      | Merge                       | Merges Level 2 record data                        | Loop Over L2 records arrays having L3 children to fetch | Add origin LinkTo                                |                                                                                                         |
| Repeat L3 links of current L2 table         | Set                         | Filters Level 3 links for current Level 2 table  | Loop Over L2 records arrays having L3 children to fetch | Merge                                            |                                                                                                         |
| Level 2 records                            | Split Out                   | Splits Level 2 records array                      | Loop Over L2 records arrays having L3 children to fetch | Merge                                            |                                                                                                         |
| Merge                                       | Merge                       | Combines Level 2 records and corresponding Level 3 links | Level 2 records, Repeat L3 links of current L2 table | Loop Over L2 Records for a given L1 field        |                                                                                                         |
| Loop Over L2 Records for a given L1 field   | Split In Batches            | Iterates over Level 2 records for each field      | Merge                                       | Aggregate to record list, Split Out L3 Potential Links |                                                                                                         |
| Aggregate to record list                     | Aggregate                   | Aggregates Level 2 records back to list           | Loop Over L2 Records for a given L1 field   | Loop Over L2 records arrays having L3 children to fetch |                                                                                                         |
| Split Out L3 Potential Links                 | Split Out                   | Splits Level 3 links array                         | Loop Over L2 Records for a given L1 field   | Prepare HTTP Call                               |                                                                                                         |
| Prepare HTTP Call                            | Set                         | Prepares parameters for Level 3 HTTP request      | Split Out L3 Potential Links                 | Get L3 Linked records without reverse Link To fields |                                                                                                         |
| Get L3 Linked records without reverse Link To fields | HTTP Request                | Fetches Level 3 linked records                      | Prepare HTTP Call                            | Preparing aggregation                           |                                                                                                         |
| Preparing aggregation                        | Set                         | Prepares Level 2 and Level 3 data for aggregation | Get L3 Linked records without reverse Link To fields | Aggregate1                                      |                                                                                                         |
| Aggregate1                                  | Aggregate                   | Aggregates Level 2 field names and Level 3 records | Preparing aggregation                        | Code to single field property                   |                                                                                                         |
| Code to single field property                | Code                        | Creates single JSON field combining Level 3 data  | Aggregate1                                  | Merge with original field property              |                                                                                                         |
| Merge with original field property           | Set                         | Merges new Level 3 fields with original Level 2 record fields | Code to single field property                | Loop Over L2 Records for a given L1 field        |                                                                                                         |
| Add origin LinkTo                            | Set                         | Adds origin link field name                        | Merge1                                      | Insert linked records to originRecord & Clean Field Name |                                                                                                         |
| Insert linked records to originRecord & Clean Field Name | Code                        | Inserts linked records into original target record | Add origin LinkTo                            | If markdown to html                             |                                                                                                         |
| If markdown to html                         | If                          | Checks if rich text conversion to HTML is requested | Insert linked records to originRecord & Clean Field Name | Rich Text Fields list, Merge2                   |                                                                                                         |
| Rich Text Fields list                       | Set                         | Lists all rich text field names across schema      | If markdown to html                          | Markdown to Html                                | ## Converts Airtable Rich Text (pseudo-markdown) to HTML  
Requires `marked` npm package. Can be deleted if not needed.                                           |
| Markdown to Html                            | Code                        | Converts rich text fields from markdown to HTML    | Rich Text Fields list                        | Merge2                                          |                                                                                                         |
| Merge2                                      | Merge                       | Merges record data and comments into final output  | Rename comments, Markdown to Html or Insert linked records to originRecord & Clean Field Name | -                                               |                                                                                                         |
| Sticky Note (multiple)                      | Sticky Note                 | Notes on various logical blocks                     | -                                          | -                                               | See relevant sticky note content in block descriptions.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**
   - Type: Execute Workflow Trigger
   - No parameters; acts as entry point.

2. **Create Inputs Node**
   - Type: Set
   - Assign these variables from incoming JSON:
     - `base_id` (string)
     - `table_id` (string)
     - `record_id` (string)
     - `level_3` (array)
     - Pass through other fields.
   - Connect trigger node output to this node.

3. **Retrieve Target Record**
   - Create Airtable node named "Airtable - Get Target Record"
   - Operation: Get Record by ID
   - Use credentials with Airtable API token.
   - Set Base ID: `={{ $json.base_id }}`
   - Set Table ID: `={{ $json.table_id }}`
   - Set Record ID: `={{ $json.record_id }}`
   - Connect Inputs node output here.

4. **Fetch Comments for Record**
   - Create HTTP Request node "Get Airtable Comments"
   - Method: GET
   - URL: `https://api.airtable.com/v0/{{$json.base_id}}/{{$json.table_id}}/{{$json.record_id}}/comments`
   - Authentication: Airtable API Token credentials
   - Connect Inputs node output here.

5. **Rename Comments Field**
   - Create Set node "Rename comments"
   - Set field `AirtableComments` to `{{$json.comments}}`
   - Connect "Get Airtable Comments" output here.

6. **Get Airtable Base Schema**
   - Create Airtable node "Airtable Schema"
   - Operation: Get Schema of Base
   - Base ID: `={{ $json.base_id }}`
   - Use Airtable API token
   - Connect output of "Airtable - Get Target Record" here.

7. **Filter Schema for Main Table**
   - Create Filter node "Filter Main Table"
   - Condition: `id` equals `={{ $json.table_id }}`
   - Connect "Airtable Schema" output here.

8. **Split Out Fields**
   - Create Split Out node "Split Out Fields"
   - Field to split: `fields`
   - Connect "Filter Main Table" output here.

9. **Filter Link To Fields**
   - Create Filter node "Filter Link To fields"
   - Condition: field `type` equals `multipleRecordLinks`
   - Connect "Split Out Fields" output here.

10. **Add Inverse LinkTo Field Name**
    - Create Set node "Add Inverse LinkTo Field name"
    - Assign `inverseFieldName` using expression to find inverse link field name in schema for each linked field.
    - Connect "Filter Link To fields" output here.

11. **Get Level 2 Linked Records**
    - Create HTTP Request node "Get Linked records without Link To fields Except for Level 3"
    - Method: POST
    - URL pattern: `https://api.airtable.com/v0/{{ $json.base_id }}/{{ $json.options.linkedTableId }}/listRecords`
    - Body: filter records linked to the target record; select fields excluding link fields except those in Level 3 input.
    - Connect "Add Inverse LinkTo Field name" output here.

12. **Check if Level 3 Fetching Needed**
    - Create If node "If Has Level 3"
    - Condition: Check if Level 3 fields exist in schema AND linked records count > 0.
    - True branch: Connect to "Loop Over L2 records arrays having L3 children to fetch"
    - False branch: Connect to "Merge1"

13. **Loop Over L2 Records Arrays Having L3 Children**
    - Create Split In Batches node "Loop Over L2 records arrays having L3 children to fetch"
    - Connect "If Has Level 3" true output here.

14. **Merge L2 Data**
    - Create Merge node "Merge1"
    - Connect "If Has Level 3" false output and "Loop Over L2 records arrays having L3 children to fetch" output here.

15. **Repeat L3 Links of Current L2 Table**
    - Create Set node "Repeat L3 links of current L2 table"
    - Assign filtered Level 3 links relevant for current L2 table.
    - Connect "Loop Over L2 records arrays having L3 children to fetch" output here.

16. **Split Level 2 Records**
    - Create Split Out node "Level 2 records"
    - Field to split: `records`
    - Connect "Loop Over L2 records arrays having L3 children to fetch" output here.

17. **Merge Level 2 Records and L3 Links**
    - Create Merge node "Merge"
    - Mode: Combine
    - Inputs: from "Level 2 records" and "Repeat L3 links of current L2 table"
    - Connect outputs accordingly.

18. **Loop Over Each Level 2 Record for a Given Field**
    - Create Split In Batches node "Loop Over L2 Records for a given L1 field"
    - Connect "Merge" output here.

19. **Aggregate Level 2 Records**
    - Create Aggregate node "Aggregate to record list"
    - Aggregate fields as needed.
    - Connect "Loop Over L2 Records for a given L1 field" output here.

20. **Split Out Level 3 Potential Links**
    - Create Split Out node "Split Out L3 Potential Links"
    - Field to split: `level_3_links`
    - Connect "Loop Over L2 Records for a given L1 field" output here.

21. **Prepare HTTP Call for Level 3 Records**
    - Create Set node "Prepare HTTP Call"
    - Assign parameters for Level 3 record fetch: table IDs, field IDs, etc.
    - Connect "Split Out L3 Potential Links" output here.

22. **Get Level 3 Linked Records**
    - Create HTTP Request node "Get L3 Linked records without reverse Link To fields"
    - POST request to Airtable API to fetch Level 3 records excluding reverse link fields.
    - Connect "Prepare HTTP Call" output here.

23. **Prepare Aggregation for Level 3 Data**
    - Create Set node "Preparing aggregation"
    - Prepare Level 2 and Level 3 data for aggregation.
    - Connect "Get L3 Linked records without reverse Link To fields" output here.

24. **Aggregate Level 3 Data**
    - Create Aggregate node "Aggregate1"
    - Aggregate Level 2 field names and Level 3 records.
    - Connect "Preparing aggregation" output here.

25. **Convert Aggregated Data to Single Field Property**
    - Create Code node "Code to single field property"
    - JavaScript converts aggregated arrays into a single JSON object with correct field assignments.
    - Connect "Aggregate1" output here.

26. **Merge with Original Level 2 Record Fields**
    - Create Set node "Merge with original field property"
    - Combine new Level 3 data with original Level 2 record fields.
    - Connect "Code to single field property" output here.
    - Connect output back to "Loop Over L2 Records for a given L1 field" for iteration.

27. **Add Origin LinkTo Field Name**
    - Create Set node "Add origin LinkTo"
    - Assigns the origin link field name for final record assembly.
    - Connect "Merge1" output here.

28. **Insert Linked Records into Origin Record**
    - Create Code node "Insert linked records to originRecord & Clean Field Name"
    - Inserts all linked records into the original target record JSON structure.
    - Connect "Add origin LinkTo" output here.

29. **Conditional Rich Text Conversion**
    - Create If node "If markdown to html"
    - Checks if input parameter `to_html` is true.
    - True branch: Connect to "Rich Text Fields list"
    - False branch: Connect to "Merge2"

30. **List Rich Text Fields**
    - Create Set node "Rich Text Fields list"
    - Collects all rich text field names from the Airtable schema.
    - Connect "If markdown to html" true output here.

31. **Convert Markdown to HTML**
    - Create Code node "Markdown to Html"
    - Uses `marked` npm package to convert rich text fields from pseudo-markdown to HTML recursively.
    - Connect "Rich Text Fields list" output here.

32. **Merge Final Data and Comments**
    - Create Merge node "Merge2"
    - Combine the final assembled record (with or without HTML conversion) and renamed comments.
    - Connect outputs from "Rename comments" and either "Markdown to Html" or "Insert linked records to originRecord & Clean Field Name".

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow requires an Airtable API token with full access to the relevant bases and tables.      | Credentials setup in n8n for Airtable API Token.                                                              |
| The rich text to HTML conversion uses the `marked` npm package; ensure it's installed in your n8n.  | [marked npm package](https://www.npmjs.com/package/marked)                                                    |
| Input JSON example for triggering the workflow is provided in the sticky note for the `Inputs` node. | See Sticky Note near `Inputs` node for example input JSON.                                                    |
| Rate limits and paging may affect performance when fetching large sets of linked records.            | Recommend batching usage and error handling for API limits.                                                   |
| The workflow includes detailed comments in sticky notes explaining multi-level relationship handling.| Sticky Notes are placed near logical blocks for clarity.                                                      |
| Refer to Airtable API docs for advanced `filterByFormula` usage and schema details.                   | [Airtable API Documentation](https://airtable.com/api)                                                        |

---

**Disclaimer:**  
This documentation is based exclusively on an automated n8n workflow designed for lawful and authorized data retrieval from Airtable. It contains no illegal or protected content and respects all applicable content policies.