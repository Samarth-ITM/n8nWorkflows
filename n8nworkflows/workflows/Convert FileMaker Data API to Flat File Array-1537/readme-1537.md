Convert FileMaker Data API to Flat File Array

https://n8nworkflows.xyz/workflows/convert-filemaker-data-api-to-flat-file-array-1537


# Convert FileMaker Data API to Flat File Array

### 1. Workflow Overview

This workflow titled **"Convert FileMaker Data API to Flat File Array"** is designed to transform a nested FileMaker Data API response into a simplified, flat array of records. The main use case is to handle data returned from the FileMaker Data API, which typically includes metadata and nested data structures, and extract only the relevant field data for easier downstream processing or exporting.

The workflow consists of the following logical blocks:

- **1.1 Data Simulation Block:** Simulates a FileMaker Data API response containing metadata and an array of records with nested `fieldData`.
- **1.2 Data Extraction Block:** Processes the simulated response to isolate the array of records.
- **1.3 Flattening Block:** Extracts the `fieldData` object from each record, producing a flat array of simple JSON objects representing each contact.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Simulation Block

- **Overview:**  
  This block simulates a FileMaker Data API response with sample contact records. It serves as the data input source for the workflow, allowing testing and demonstration without needing a live API call.

- **Nodes Involved:**  
  - `FileMaker Data API Contacts`

- **Node Details:**  
  - **Node Name:** FileMaker Data API Contacts  
  - **Type:** Function  
  - **Technical Role:** Generates a mock JSON response mimicking FileMaker's Data API output with metadata and an array of records.  
  - **Configuration Choices:**  
    - Returns a single item with a `json` object containing a `response` key.  
    - The `response.data` array holds multiple records, each with nested `fieldData`, `portalData`, `recordId`, and `modId`.  
    - Includes `dataInfo` metadata such as database name, layout, record counts.  
  - **Key Expressions / Variables:** None (static data)  
  - **Input Connections:** None (starting point)  
  - **Output Connections:** Outputs to `FileMaker response.data` node.  
  - **Version-specific Requirements:** Compatible with n8n Function node version 1.  
  - **Potential Failure Cases:** None, as this is static data generation.  
  - **Sub-workflow Reference:** None.

#### 2.2 Data Extraction Block

- **Overview:**  
  This block isolates the `response.data` array from the nested JSON output of the previous node, effectively extracting the list of records from the full response.

- **Nodes Involved:**  
  - `FileMaker response.data`

- **Node Details:**  
  - **Node Name:** FileMaker response.data  
  - **Type:** Item Lists  
  - **Technical Role:** Splits the input data array into individual items for further processing.  
  - **Configuration Choices:**  
    - Configured to split out the field located at `response.data` (using an expression `=response.data`).  
    - This extracts the entire array of contact records as separate items.  
  - **Key Expressions / Variables:** Expression `=response.data` used to select the array from input JSON.  
  - **Input Connections:** Connected from `FileMaker Data API Contacts` node.  
  - **Output Connections:** Outputs to `Return item.fieldData` node.  
  - **Version-specific Requirements:** Compatible with n8n Item Lists node, version 1.  
  - **Potential Failure Cases:**  
    - If the input JSON structure does not contain `response.data`, this node will fail or output empty results.  
    - If input is empty or not an array, splitting may produce no items.  
  - **Sub-workflow Reference:** None.

#### 2.3 Flattening Block

- **Overview:**  
  This block extracts the `fieldData` property from each record item, flattening the nested structure into simple JSON objects containing only field values for each contact.

- **Nodes Involved:**  
  - `Return item.fieldData`

- **Node Details:**  
  - **Node Name:** Return item.fieldData  
  - **Type:** Function Item  
  - **Technical Role:** Transforms each incoming item by returning only its `fieldData` object.  
  - **Configuration Choices:**  
    - JavaScript function code: `return item.fieldData;`  
    - This replaces each item with its `fieldData` content, removing other metadata.  
  - **Key Expressions / Variables:** Uses `item.fieldData` to access contact fields.  
  - **Input Connections:** Connected from `FileMaker response.data` node.  
  - **Output Connections:** None (end of workflow)  
  - **Version-specific Requirements:** Compatible with n8n Function Item node version 1.  
  - **Potential Failure Cases:**  
    - If any item lacks the `fieldData` property, output will be `undefined` or cause error.  
    - Input that is not an object or malformed may cause expression failure.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                   | Input Node(s)             | Output Node(s)           | Sticky Note                                              |
|---------------------------|--------------------|---------------------------------|---------------------------|--------------------------|----------------------------------------------------------|
| FileMaker Data API Contacts | Function           | Simulate FileMaker API response | (none)                    | FileMaker response.data   | Basis workflow to convert FileMaker Data API ...         |
| FileMaker response.data    | Item Lists         | Extract `response.data` array    | FileMaker Data API Contacts | Return item.fieldData     |                                                          |
| Return item.fieldData      | Function Item      | Return only the `fieldData` part| FileMaker response.data    | (none)                   |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Function Node** named `FileMaker Data API Contacts`:
   - Purpose: Simulate FileMaker Data API response.
   - Paste the JavaScript code returning the static JSON response with `response.data` array containing multiple records.
   - No input connections.
   - No credentials required.
   - Version: Function node, version 1.

2. **Create an Item Lists Node** named `FileMaker response.data`:
   - Purpose: Extract the array of records from the simulated response.
   - In the "Field To Split Out" parameter, enter the expression: `=response.data`.
   - Connect the output of `FileMaker Data API Contacts` to this node.
   - Version: Item Lists node, version 1.

3. **Create a Function Item Node** named `Return item.fieldData`:
   - Purpose: Flatten each record item by returning only its `fieldData`.
   - JavaScript code: `return item.fieldData;`
   - Connect the output of `FileMaker response.data` to this node.
   - Version: Function Item node, version 1.

4. **Save the workflow and run it**:
   - The final output will be an array of flat JSON objects representing contacts, without nested metadata.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                          |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| The workflow uses static sample data to simulate FileMaker Data API output for demonstration and testing.     | Provided code in `FileMaker Data API Contacts` node |
| This approach helps avoid API call complexity and facilitates quick prototyping of data transformations.       |                                         |
| For integration with a real FileMaker Data API, replace the simulation node with an HTTP Request node to fetch live data. |                                         |

---

This documentation fully describes the workflow structure and operation, enabling reproduction, modification, and troubleshooting without the original JSON file.