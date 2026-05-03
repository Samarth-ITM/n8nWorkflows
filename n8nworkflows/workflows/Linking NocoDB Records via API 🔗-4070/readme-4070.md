Linking NocoDB Records via API 🔗

https://n8nworkflows.xyz/workflows/linking-nocodb-records-via-api----4070


# Linking NocoDB Records via API 🔗

### 1. Workflow Overview

This workflow automates the process of linking records between two tables in NocoDB via API, facilitating many-to-many relationships. It fetches the necessary record IDs, retrieves current linked records, appends a new link, and updates the target record in one atomic API request. The workflow is designed for users who want to create or maintain relational links between NocoDB tables without manual API calls or coding, relying instead on n8n’s visual automation.

**Target Use Case:**  
Linking records from a "source" table (many side) to a "target" table (one or many side) in NocoDB, ideal for scenarios like associating multiple scenes to video productions or other many-to-many database relationships.

**Logical Blocks:**

- **1.1 Input Initialization:** Manual trigger and variable setup defining target and source record and table IDs.  
- **1.2 Data Retrieval:** Fetching detailed data from the target and source tables using NocoDB API nodes.  
- **1.3 Metadata Retrieval:** Obtaining metadata about the target table to identify the correct link field for updating.  
- **1.4 Linking Operation:** Sending a POST request to NocoDB API to link the source record(s) to the target record.  
- **1.5 Documentation and Guidance:** Sticky notes providing instructions, API references, and video tutorial links.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

**Overview:**  
Starts the workflow manually and sets key variables such as NocoDB URL, table IDs, and record IDs for source and target tables.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set Variables

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to initiate workflow execution manually for testing or ad hoc use.  
  - Configuration: Default manual trigger, no parameters.  
  - Input: None  
  - Output: Starts workflow, passes empty data to next node.  
  - Edge Cases: None typical; user must manually trigger.

- **Set Variables**  
  - Type: Set  
  - Role: Defines and sets key variables used in the workflow, including NocoDB URL, target/source table IDs, and record IDs.  
  - Configuration: Four string variables assigned:
    - `my_nocodb`: User’s NocoDB base URL (e.g., nocodb.myserver.com)  
    - `target_table_id`: ID of the target table (destination)  
    - `target_table_row_id`: Record ID in the target table to link to  
    - `source_table_row_id`: Record ID in the source table to link from  
  - Input: Trigger from manual node  
  - Output: Passes variables to next node  
  - Edge Cases: Variables must be set correctly; invalid or missing IDs will cause downstream failures.  
  - Sticky Note5 visually covers this node, emphasizing "Set These!" variables.

#### 1.2 Data Retrieval

**Overview:**  
Fetches record data from both the target and source tables using these IDs, preparing the data for the linking operation.

**Nodes Involved:**  
- Grab Target Table Row  
- Get Source Table Row

**Node Details:**

- **Grab Target Table Row**  
  - Type: NocoDB node (v3)  
  - Role: Retrieves the target table record by ID to obtain its current data and linked entries.  
  - Configuration:  
    - Table: Fixed ID `"mc5ihmltcqfco2v"` (target table)  
    - Project ID: `"p82se8ug6ui5lt0"` (NocoDB project context)  
    - Record ID: Expression `={{ $json.target_table_row_id }}` (from Set Variables)  
    - Authentication: NocoDB API token credential  
  - Input: From Set Variables  
  - Output: Target record data to next node  
  - Edge Cases: Invalid record ID or permission errors on API token.

- **Get Source Table Row**  
  - Type: NocoDB node (v3)  
  - Role: Retrieves the source table record by ID to get its record ID for linking.  
  - Configuration:  
    - Table: Fixed ID `"mm94d1xdbfrn1oe"` (source table)  
    - Project ID: `"p82se8ug6ui5lt0"`  
    - Record ID: Expression `={{ $('Set Variables').item.json.source_table_row_id }}`  
    - Authentication: NocoDB API token credential  
  - Input: From Grab Target Table Row (via connection chain)  
  - Output: Source record data for use in POST body  
  - Edge Cases: Same as above, invalid ID or API issues.

#### 1.3 Metadata Retrieval

**Overview:**  
Queries the meta API of NocoDB to obtain schema details for the target table, specifically to identify the correct link field ID required for the linking API call.

**Nodes Involved:**  
- Get Target Table Meta Data

**Node Details:**

- **Get Target Table Meta Data**  
  - Type: HTTP Request (v4.2)  
  - Role: Retrieves metadata for the target table including columns and link field IDs to know where to post the link.  
  - Configuration:  
    - URL: `https://{{ $('Set Variables').item.json.my_nocodb }}/api/v2/meta/tables/{{ $('Set Variables').item.json.target_table_id }}`  
    - Method: GET (default)  
    - Headers: Accept `application/json`  
    - Authentication: NocoDB API token credential (predefined)  
  - Input: From Get Source Table Row  
  - Output: Metadata JSON to next node  
  - Edge Cases: Network timeouts, invalid URL, authentication failure  
  - Notes: Uses dynamic expressions to build URL with variables.

#### 1.4 Linking Operation

**Overview:**  
Performs the actual linking by sending a POST request to the NocoDB API endpoint responsible for linking records, passing the source record ID in the request body.

**Nodes Involved:**  
- Link Record from Source to Target

**Node Details:**

- **Link Record from Source to Target**  
  - Type: HTTP Request (v4.2)  
  - Role: Sends a POST request to link the source record(s) to the target record in NocoDB via the links API.  
  - Configuration:  
    - URL:  
      `https://{{ $('Set Variables').item.json.my_nocodb }}/api/v2/tables/{{ $('Set Variables').item.json.target_table_id }}/links/{{ $('Get Target Table Meta Data').item.json.columns[11].id }}/records/{{ $('Grab Target Table Row').item.json['Record ID'] }}`  
      (Note: The column index `[11]` assumes the link field is at position 12 in the columns array.)  
    - Method: POST  
    - Headers: `accept: application/json`, `content-type: application/json`  
    - Body (JSON): `[ { "Id": {{ $('Get Source Table Row').item.json['Record ID'] }} } ]`  
    - Authentication: NocoDB API token credential  
  - Input: From Get Target Table Meta Data  
  - Output: API response confirming link update  
  - Edge Cases:  
    - Incorrect link field index or ID leads to invalid API endpoint  
    - API errors if record IDs are invalid or permission denied  
    - Ensure correct JSON formatting to avoid body parse errors

#### 1.5 Documentation and Guidance

**Overview:**  
Sticky notes provide users with essential references, API endpoints, explanations of many-to-many relationships, and video guides for setup.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5

**Node Details:**

- **Sticky Note**  
  - Content: API endpoint template, NocoDB Data API and Meta API links  
- **Sticky Note1**  
  - Content: Explanation of the target table concept, many-to-many relationship example  
- **Sticky Note2**  
  - Content: Explanation of the source table where link field exists  
- **Sticky Note3**  
  - Content: Detailed example of API POST URL and body format for linking  
- **Sticky Note4**  
  - Content: Video guide URL https://youtu.be/-srzNushUsk  
- **Sticky Note5**  
  - Content: Reminder to set variables correctly

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                          | Input Node(s)                | Output Node(s)                  | Sticky Note                                 |
|----------------------------|---------------------|----------------------------------------|-----------------------------|--------------------------------|---------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Start workflow manually                 | None                        | Set Variables                  |                                             |
| Set Variables              | Set                 | Sets NocoDB URL, table and record IDs | When clicking ‘Test workflow’ | Grab Target Table Row          | "Set These!" reminder                        |
| Grab Target Table Row      | NocoDB node         | Fetch target record data by ID          | Set Variables               | Get Source Table Row           |                                             |
| Get Source Table Row       | NocoDB node         | Fetch source record data by ID          | Grab Target Table Row       | Get Target Table Meta Data     |                                             |
| Get Target Table Meta Data | HTTP Request        | Get metadata of target table            | Get Source Table Row        | Link Record from Source to Target |                                             |
| Link Record from Source to Target | HTTP Request        | Perform POST to link source record to target record | Get Target Table Meta Data | None                           |                                             |
| Sticky Note                | Sticky Note         | API references and endpoints             | None                        | None                          | Covers workflow overview and references      |
| Sticky Note1               | Sticky Note         | Explains target table and many-to-many | None                        | None                          | Explains target table relationship           |
| Sticky Note2               | Sticky Note         | Explains source table concept            | None                        | None                          | Explains source table relationship           |
| Sticky Note3               | Sticky Note         | Detailed API POST example                 | None                        | None                          | Detailed API call example                      |
| Sticky Note4               | Sticky Note         | Video guide link                         | None                        | None                          | Contains video guide URL https://youtu.be/-srzNushUsk |
| Sticky Note5               | Sticky Note         | Reminder to set variables                 | None                        | None                          | "Set These!" variables reminder               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters.

2. **Create a Set node**  
   - Name: `Set Variables`  
   - Connect input from `When clicking ‘Test workflow’`  
   - Add string variables:  
     - `my_nocodb`: Your NocoDB base URL (e.g., nocodb.myserver.com)  
     - `target_table_id`: Target table ID (e.g., from NocoDB UI)  
     - `target_table_row_id`: Target record ID to link to (e.g., 5)  
     - `source_table_row_id`: Source record ID to link from (e.g., 23)

3. **Create a NocoDB node to get target record**  
   - Name: `Grab Target Table Row`  
   - Connect input from `Set Variables`  
   - Parameters:  
     - Table: Enter your target table ID  
     - Project ID: Your NocoDB project ID  
     - Record ID: Set with expression `={{ $json.target_table_row_id }}`  
   - Credentials: Attach your NocoDB API token credential.

4. **Create a NocoDB node to get source record**  
   - Name: `Get Source Table Row`  
   - Connect input from `Grab Target Table Row`  
   - Parameters:  
     - Table: Enter your source table ID  
     - Project ID: Same as above  
     - Record ID: Expression `={{ $('Set Variables').item.json.source_table_row_id }}`  
   - Credentials: Same NocoDB API token credential.

5. **Create an HTTP Request node to get target table metadata**  
   - Name: `Get Target Table Meta Data`  
   - Connect input from `Get Source Table Row`  
   - Parameters:  
     - URL: Expression `=https://{{ $('Set Variables').item.json.my_nocodb }}/api/v2/meta/tables/{{ $('Set Variables').item.json.target_table_id }}`  
     - Method: GET  
     - Headers: Add `accept: application/json`  
   - Authentication: Use NocoDB API token credential.

6. **Create an HTTP Request node to link records**  
   - Name: `Link Record from Source to Target`  
   - Connect input from `Get Target Table Meta Data`  
   - Parameters:  
     - URL: Expression `=https://{{ $('Set Variables').item.json.my_nocodb }}/api/v2/tables/{{ $('Set Variables').item.json.target_table_id }}/links/{{ $('Get Target Table Meta Data').item.json.columns[11].id }}/records/{{ $('Grab Target Table Row').item.json['Record ID'] }}`  
       *Note:* Adjust `[11]` to the index of your link column in metadata columns array.  
     - Method: POST  
     - Headers:  
       - `accept: application/json`  
       - `content-type: application/json`  
     - Body Content Type: JSON  
     - Body: Expression `[ { "Id": {{ $('Get Source Table Row').item.json['Record ID'] }} } ]`  
   - Authentication: Use NocoDB API token credential.

7. **Set up credentials**  
   - Create a credential of type "NocoDB API Token" with a valid API token that has permission to read/write the involved tables.

8. **Test the workflow**  
   - Manually trigger the workflow.  
   - Verify each node returns expected data.  
   - Confirm that the link is created in NocoDB by checking the target record’s linked field.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| NocoDB Many to Many Link Record API POST example: `http://localhost:8080/api/v2/tables/{tableId}/links/{linkFieldId}/records/{recordId}` | Sticky Note with API endpoint template            |
| NocoDB Data API official docs: https://data-apis-v2.nocodb.com/#tag/Table-Records/operation/db-data-table-row-nested-list | Sticky Note with API documentation link           |
| NocoDB Meta API official docs: https://meta-apis-v2.nocodb.com/                                      | Sticky Note with API documentation link           |
| Video Guide for this workflow: https://youtu.be/-srzNushUsk                                         | Sticky Note4 providing visual setup assistance    |
| Many-to-Many linking explanation: Target and Source tables roles and relationship examples           | Sticky Note1 and Sticky Note2                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all current content policies without any illegal, offensive, or protected elements. All data handled is legal and public.