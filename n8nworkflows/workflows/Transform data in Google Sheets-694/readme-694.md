Transform data in Google Sheets

https://n8nworkflows.xyz/workflows/transform-data-in-google-sheets-694


# Transform data in Google Sheets

### 1. Workflow Overview

This workflow demonstrates how to interact with a Google Sheets spreadsheet by performing four core operations: appending data, looking up data, updating data, and reading data. It is designed to be triggered manually and operates sequentially on a specified spreadsheet. The workflow is useful for scenarios where managing and manipulating tabular data in Google Sheets is required, such as property rental management or inventory tracking.

Logical blocks in this workflow:

- **1.1 Input Data Generation:** Generates or receives the data to be appended to the Google Sheet.
- **1.2 Append Data to Google Sheets:** Adds new rows with generated data.
- **1.3 Lookup Data in Google Sheets:** Searches the sheet for rows matching a specific criterion.
- **1.4 Data Transformation:** Modifies the looked-up data (e.g., updates rent values).
- **1.5 Update Data in Google Sheets:** Updates existing rows based on transformed data.
- **1.6 Read Data from Google Sheets:** Retrieves and outputs data from the sheet after updates.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Data Generation

- **Overview:**  
  This block generates sample data representing a property listing, including ID, Name, Rent, and City. It simulates incoming data that would be appended to the Google Sheet. In other contexts, this data could come from a webhook or any other source.

- **Nodes Involved:**  
  - Set

- **Node Details:**

  - **Set**  
    - Type & Role: Set node to define and format data fields before insertion.  
    - Configuration:  
      - Generates a random integer ID (`Math.floor(Math.random()*1000)`).  
      - Sets static string values: Name = "John's Place", Rent = "$1,000", City = "Berlin".  
      - Keeps only these set values, discarding any previous input.  
    - Expressions:  
      - ID: `={{Math.floor(Math.random()*1000)}}`  
      - Other fields: static strings.  
    - Input: Manual trigger node output (empty dataset initially).  
    - Output: JSON object with keys ID, Name, Rent, City.  
    - Failure Cases: Expression errors unlikely but could occur if JavaScript expression syntax changes in future versions.  
    - Version: Compatible with n8n v1.x.  
    - No sub-workflow reference.

#### 2.2 Append Data to Google Sheets

- **Overview:**  
  Appends the generated property data as a new row in the specified Google Sheet.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type & Role: Google Sheets node performing the "append" operation.  
    - Configuration:  
      - Spreadsheet ID: `"1remFwo--5ehUgIU7UUndKldPI0Xm93e1T3DldD9GOg0"`.  
      - Range: `"A:D"` (columns A to D).  
      - Value Input Mode: `"USER_ENTERED"` (interprets input as if typed by user, allowing formulas).  
      - Authentication: OAuth2 via configured credential `"google-sheet"`.  
      - Operation: Append new row.  
    - Input: Output from Set node (data to append).  
    - Output: Confirmation of append operation, including row info.  
    - Failure Cases: Authentication errors, invalid spreadsheet ID, permission denied, network timeouts.  
    - Version: Requires OAuth2-compatible Google Sheets node (n8n v1.x).  
    - No sub-workflow reference.

#### 2.3 Lookup Data in Google Sheets

- **Overview:**  
  Searches the Google Sheet for all rows where the "City" column matches "Berlin". Returns matching rows for further processing.

- **Nodes Involved:**  
  - Google Sheets1

- **Node Details:**

  - **Google Sheets1**  
    - Type & Role: Google Sheets node performing a "lookup" operation.  
    - Configuration:  
      - Spreadsheet ID: same as above.  
      - Range: `"A:D"`.  
      - Lookup Column: `"City"`.  
      - Lookup Value: `"Berlin"`.  
      - Return All Matches: `true` (returns all rows matching criteria).  
      - Value Render Mode: `"UNFORMATTED_VALUE"` (returns raw cell values).  
      - Authentication: OAuth2 credential `"google-sheet"`.  
      - Operation: Lookup rows.  
    - Input: Output from previous append operation (though logically it could be independent).  
    - Output: Array of matching rows where City = Berlin.  
    - Failure Cases: Authentication errors, spreadsheet access issues, no matches found (empty output).  
    - Version: OAuth2 support required.  
    - No sub-workflow reference.

#### 2.4 Data Transformation

- **Overview:**  
  Increments the Rent value by 100 for each property located in Berlin, preparing updated data for the Google Sheets update operation.

- **Nodes Involved:**  
  - Set1

- **Node Details:**

  - **Set1**  
    - Type & Role: Set node used to modify data fields for update.  
    - Configuration:  
      - Updates "Rent" field by adding 100 to the existing rent value retrieved from `Google Sheets1`.  
      - Copies "ID", "Name", and "City" unchanged from the lookup result.  
      - Keeps only these set fields.  
    - Expressions:  
      - Rent: `={{$node["Google Sheets1"].json["Rent"] + 100}}` (assumes Rent is numeric).  
      - ID, Name, City: copied as is.  
    - Input: Output from Google Sheets1 (lookup results).  
    - Output: Updated JSON with new Rent and existing other fields.  
    - Failure Cases: Expression failure if Rent is not numeric or missing, or if multiple rows returned (expression assumes single object).  
    - Version: n8n v1.x.  
    - No sub-workflow reference.

#### 2.5 Update Data in Google Sheets

- **Overview:**  
  Updates existing rows in the Google Sheet with the new rent values, mapping rows by the "ID" column.

- **Nodes Involved:**  
  - Google Sheets2

- **Node Details:**

  - **Google Sheets2**  
    - Type & Role: Google Sheets node performing the "update" operation.  
    - Configuration:  
      - Spreadsheet ID: same as above.  
      - Range: `"A:D"`.  
      - Key column for row identification: `"ID"`.  
      - Value Input Mode: `"USER_ENTERED"`.  
      - Value Render Mode: `"UNFORMATTED_VALUE"`.  
      - Authentication: OAuth2 credential `"google-sheet"`.  
      - Operation: Update rows based on key.  
    - Input: Updated data from Set1 node.  
    - Output: Confirmation of updated rows.  
    - Failure Cases: Authentication failures, mismatch on ID keys causing no update, invalid data types, network issues.  
    - Version: n8n v1.x.  
    - No sub-workflow reference.

#### 2.6 Read Data from Google Sheets

- **Overview:**  
  Reads and returns the current data from columns A to D of the Google Sheet, reflecting any appended or updated data.

- **Nodes Involved:**  
  - Google Sheets3

- **Node Details:**

  - **Google Sheets3**  
    - Type & Role: Google Sheets node performing "read" operation.  
    - Configuration:  
      - Spreadsheet ID: same as above.  
      - Range: `"A:D"`.  
      - Value Render Mode: `"FORMATTED_VALUE"` (returns values as displayed in the sheet).  
      - Authentication: OAuth2 credential `"google-sheet"`.  
      - Operation: Read data.  
    - Input: Output from Google Sheets2 (update confirmation).  
    - Output: Full data from specified range.  
    - Failure Cases: Authentication errors, spreadsheet access issues, range invalid.  
    - Version: n8n v1.x.  
    - No sub-workflow reference.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                             | Input Node(s)               | Output Node(s)        | Sticky Note                                                                                          |
|--------------------|---------------------|---------------------------------------------|-----------------------------|-----------------------|----------------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger      | Initiates the workflow manually             | —                           | Set                   |                                                                                                    |
| Set                | Set                 | Generates data to append to Google Sheets   | On clicking 'execute'        | Google Sheets          | The Set node is used to generate data that we want to add to Google Sheets...                       |
| Google Sheets      | Google Sheets       | Appends the generated data as a new row     | Set                         | Google Sheets1         | This node will add the data from the Set node in a new row to the Google Sheet...                   |
| Google Sheets1     | Google Sheets       | Looks up rows where City = "Berlin"          | Google Sheets               | Set1                   | This node looks for a specific value in the Google Sheet and returns all matching rows...          |
| Set1               | Set                 | Updates Rent by adding 100 for Berlin rows  | Google Sheets1              | Google Sheets2          | The Set node sets the value of the rent by $100 for the houses in Berlin...                         |
| Google Sheets2     | Google Sheets       | Updates existing rows with new Rent values  | Set1                        | Google Sheets3          | This node will update the rent for the houses in Berlin with the new rent set in the previous node.|
| Google Sheets3     | Google Sheets       | Reads data from the sheet after updates     | Google Sheets2              | —                      | This node returns the information from the Google Sheet...                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "On clicking 'execute'"  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node to Generate Data**  
   - Name: "Set"  
   - Type: Set  
   - Parameters:  
     - Add Number field "ID" with value: `={{Math.floor(Math.random()*1000)}}`  
     - Add String fields:  
       - "Name" = `"John's Place"`  
       - "Rent" = `"$1,000"`  
       - "City" = `"Berlin"`  
     - Enable "Keep Only Set" option.  
   - Connect: Output of "On clicking 'execute'" → Input of "Set".

3. **Create Google Sheets Node to Append Data**  
   - Name: "Google Sheets"  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: "Append"  
     - Spreadsheet ID: `1remFwo--5ehUgIU7UUndKldPI0Xm93e1T3DldD9GOg0`  
     - Range: `A:D`  
     - Value Input Mode: "USER_ENTERED"  
   - Credentials: Set up OAuth2 credential for Google Sheets (`google-sheet`).  
   - Connect: Output of "Set" → Input of "Google Sheets".

4. **Create Google Sheets Node to Lookup Data**  
   - Name: "Google Sheets1"  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: "Lookup"  
     - Spreadsheet ID: same as above  
     - Range: `A:D`  
     - Lookup Column: `"City"`  
     - Lookup Value: `"Berlin"`  
     - Return All Matches: enabled  
     - Value Render Mode: "UNFORMATTED_VALUE"  
   - Credentials: Use same OAuth2 credential.  
   - Connect: Output of "Google Sheets" → Input of "Google Sheets1".

5. **Create Set Node to Transform Data**  
   - Name: "Set1"  
   - Type: Set  
   - Parameters:  
     - Number fields:  
       - "Rent" = `={{$node["Google Sheets1"].json["Rent"] + 100}}`  
       - "ID" = `={{$node["Google Sheets1"].json["ID"]}}`  
     - String fields:  
       - "Name" = `={{$node["Google Sheets1"].json["Name"]}}`  
       - "City" = `={{$node["Google Sheets1"].json["City"]}}`  
     - Enable "Keep Only Set".  
   - Connect: Output of "Google Sheets1" → Input of "Set1".

6. **Create Google Sheets Node to Update Data**  
   - Name: "Google Sheets2"  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: "Update"  
     - Spreadsheet ID: same as above  
     - Range: `A:D`  
     - Key Column: `"ID"`  
     - Value Input Mode: "USER_ENTERED"  
     - Value Render Mode: "UNFORMATTED_VALUE"  
   - Credentials: Use same OAuth2 credential.  
   - Connect: Output of "Set1" → Input of "Google Sheets2".

7. **Create Google Sheets Node to Read Data**  
   - Name: "Google Sheets3"  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: "Read"  
     - Spreadsheet ID: same as above  
     - Range: `A:D`  
     - Value Render Mode: "FORMATTED_VALUE"  
   - Credentials: Use same OAuth2 credential.  
   - Connect: Output of "Google Sheets2" → Input of "Google Sheets3".

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow can be modularized into separate workflows for appending, looking up, updating, or reading data as per use-case. | Workflow design tip.                                                                             |
| Refer to [Google Sheets node documentation](https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets/#google-sheets) for detailed node setup. | Official n8n Docs.                                                                              |
| Data transformation assumes Rent is numeric; if storing as string with currency symbols, parsing may be needed. | Potential data format consideration.                                                           |
| OAuth2 credentials for Google Sheets must have appropriate scopes for reading and writing spreadsheet data. | Credential setup requirement.                                                                   |
| Random ID generation here could cause collisions; in production, consider more robust unique identifiers. | Data integrity consideration.                                                                   |

---

This documentation fully captures the workflow’s structure, logic, configuration, and potential pitfalls, enabling effective reproduction and extension by users or AI agents.