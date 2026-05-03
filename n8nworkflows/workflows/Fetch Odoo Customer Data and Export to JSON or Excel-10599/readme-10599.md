Fetch Odoo Customer Data and Export to JSON or Excel

https://n8nworkflows.xyz/workflows/fetch-odoo-customer-data-and-export-to-json-or-excel-10599


# Fetch Odoo Customer Data and Export to JSON or Excel

### 1. Workflow Overview

This n8n workflow implements a REST API endpoint `/api/v1/get-customers` designed to fetch customer contact data from an Odoo instance and return it either as JSON or as an Excel (.xlsx) file. It targets users who require on-demand exports of customer data with flexible formatting for integrations, reporting, or further processing.

The workflow is logically organized into these main blocks:

- **1.1 Input Reception & Validation**: Receives HTTP requests, extracts query parameters, and validates required inputs.
- **1.2 Data Querying from Odoo**: Constructs dynamic filters based on input and queries the Odoo `res.partner` model for matching customer contacts.
- **1.3 Data Preparation**: Processes the raw data, adds metadata, and prepares the output structure.
- **1.4 Response Format Decision**: Checks if the response should be JSON or Excel based on a request parameter.
- **1.5 Excel Conversion & Response**: Converts data to Excel and responds with a downloadable file if requested.
- **1.6 JSON Response**: Responds directly with JSON data if Excel format is not requested.

These blocks are chained sequentially, with conditional branching for response format handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block exposes a webhook endpoint to receive API requests, extracts query parameters, and ensures the mandatory `name` parameter is included for filtering customer records.

- **Nodes Involved:**  
  - Receive Company Request1 (Webhook)  
  - Prepare Dynamic Filter1 (Function)  
  - Overview Note2 (Sticky Note)

- **Node Details:**

  - **Receive Company Request1**  
    - Type: Webhook  
    - Role: Entry point for HTTP GET requests at `/api/v1/get-customers`.  
    - Config: Path set to `/api/v1/get-customers`; response mode set to `responseNode` for asynchronous handling.  
    - Inputs: External HTTP request  
    - Outputs: Passes request data to the next node  
    - Failure cases: Invalid HTTP methods, network errors, malformed requests.  
    - Notes: Webhook must be publicly accessible or proxied for external calls.

  - **Prepare Dynamic Filter1**  
    - Type: Function  
    - Role: Validates input parameters and builds a filter array for querying Odoo.  
    - Config:  
      - Requires `name` parameter; returns error JSON if missing.  
      - Constructs Odoo filter `[["name", "Like", <name>]]` for partial matching on the customer’s name.  
      - Reads optional `response_format` parameter defaulting to `"json"`.  
    - Key Expressions: Uses `$json["query"]` to access query parameters.  
    - Inputs: Request JSON from webhook node.  
    - Outputs: JSON with `filters` array and `response_format`.  
    - Failure: Missing or empty `name` parameter triggers early exit with error message.

  - **Overview Note2**  
    - Type: Sticky Note  
    - Content: Explains request reception, validation logic, and mandatory parameter requirements.

---

#### 2.2 Data Querying from Odoo

- **Overview:**  
  Queries the Odoo `res.partner` model for customer contacts matching the dynamic filters prepared. Returns all matching records with selected fields.

- **Nodes Involved:**  
  - Fetch Customer  
  - Overview Note3

- **Node Details:**

  - **Fetch Customer**  
    - Type: Odoo Node  
    - Role: Fetches all matching customer records from Odoo.  
    - Config:  
      - Resource: `custom`  
      - Operation: `getAll`  
      - Custom Resource: `res.partner` (Odoo contacts model)  
      - Filters: Uses the prepared filter from previous node, specifically a "like" filter on `name`.  
      - Fields: `display_name`, `name`, `email`, `phone`, `mobile`, `parent_id`, `country_code`, `country_id`.  
      - ReturnAll: true (fetches all matching records without pagination)  
    - Credentials: Uses configured Odoo API credentials.  
    - Inputs: JSON with filters from "Prepare Dynamic Filter1".  
    - Outputs: List of customer records.  
    - Failure: Authentication errors, Odoo API downtime, invalid filter syntax, timeout for large data sets.

  - **Overview Note3**  
    - Type: Sticky Note  
    - Content: Describes query logic, notes case-sensitive "Like" search, and fields customization instructions.

---

#### 2.3 Data Preparation

- **Overview:**  
  Prepares the raw data by adding a timestamp to each record and formats the output JSON. Also handles the empty result scenario gracefully.

- **Nodes Involved:**  
  - Prepare Output Data1  
  - Overview Note4

- **Node Details:**

  - **Prepare Output Data1**  
    - Type: Function  
    - Role: Transforms Odoo records into final JSON structure, adding a generation timestamp.  
    - Config:  
      - Checks if returned data is empty; if so, returns an error JSON with a descriptive message.  
      - Maps each record to include a `report_generated_on` ISO timestamp field.  
    - Inputs: Items (customer records) from the Odoo node.  
    - Outputs: Prepared JSON data array or error object.  
    - Failure: Null or malformed input data; no matching records.

  - **Overview Note4**  
    - Type: Sticky Note  
    - Content: Summarizes data preparation and JSON object management.

---

#### 2.4 Response Format Decision

- **Overview:**  
  Determines whether the response should be JSON or Excel based on the `response_format` query parameter.

- **Nodes Involved:**  
  - Check If Excel Required1  
  - Overview Note5

- **Node Details:**

  - **Check If Excel Required1**  
    - Type: If  
    - Role: Conditional branching node that checks if the response format is exactly `"excel"`.  
    - Config: Compares the expression output of `response_format` from the filter preparation node to the string `"excel"`.  
    - Inputs: Prepared JSON data from the previous function node.  
    - Outputs:  
      - True branch: proceeds to Excel file preparation.  
      - False branch: proceeds to respond with JSON.  
    - Edge cases: Case sensitivity of `"excel"` string; missing or malformed parameter defaults to JSON.

  - **Overview Note5**  
    - Type: Sticky Note  
    - Content: Explains branching logic based on `response_format`.

---

#### 2.5 Excel Conversion & Response

- **Overview:**  
  Converts the prepared JSON data into an Excel file and responds with a binary downloadable file.

- **Nodes Involved:**  
  - Return all data for create binary file (Code)  
  - Convert to Excel1 (ConvertToFile)  
  - Respond with File1 (RespondToWebhook)  
  - Overview Notes 6,7,8

- **Node Details:**

  - **Return all data for create binary file**  
    - Type: Code  
    - Role: Passes all incoming items downstream for file conversion.  
    - Config: Returns all input data without modification.  
    - Inputs: Data from the "Check If Excel Required1" node (true branch).  
    - Outputs: Data array for Excel conversion.

  - **Convert to Excel1**  
    - Type: ConvertToFile  
    - Role: Converts JSON data to `.xlsx` Excel file format.  
    - Config: Operation set to `xlsx`. No additional options configured.  
    - Inputs: JSON array to be serialized into Excel.  
    - Outputs: Binary Excel file data.  
    - Failure: Conversion errors if data is malformed.

  - **Respond with File1**  
    - Type: RespondToWebhook  
    - Role: Sends the generated Excel file as binary response to the original HTTP request.  
    - Config: RespondWith set to binary data.  
    - Inputs: Binary Excel file.  
    - Outputs: HTTP response with file download.  
    - Failure: Network or webhook response errors.

  - **Overview Note6**  
    - Prepares data for binary file creation.

  - **Overview Note7**  
    - Indicates Excel file creation step.

  - **Overview Note8**  
    - Notes response node that sends Excel file back to client.

---

#### 2.6 JSON Response

- **Overview:**  
  Sends the prepared JSON data directly back to the caller when the requested response format is not Excel.

- **Nodes Involved:**  
  - Respond with JSON1  
  - Overview Note9

- **Node Details:**

  - **Respond with JSON1**  
    - Type: RespondToWebhook  
    - Role: Sends JSON array or error message as HTTP response.  
    - Config: RespondWith set to `allIncomingItems` to include all data items.  
    - Inputs: JSON data from "Prepare Output Data1" via the false branch of the If node.  
    - Outputs: HTTP JSON response.  
    - Failure: Network errors or invalid JSON data.

  - **Overview Note9**  
    - Describes JSON response node usage.

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                          | Input Node(s)             | Output Node(s)                          | Sticky Note                                                                                                       |
|------------------------------|----------------------|----------------------------------------|---------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Receive Company Request1      | Webhook              | Receives API HTTP request               | None                      | Prepare Dynamic Filter1                 |                                                                                                                   |
| Prepare Dynamic Filter1       | Function             | Validates input & builds Odoo filters   | Receive Company Request1   | Fetch Customer                         |                                                                                                                   |
| Fetch Customer               | Odoo                 | Queries Odoo for matching customers     | Prepare Dynamic Filter1    | Prepare Output Data1                   |                                                                                                                   |
| Prepare Output Data1          | Function             | Adds metadata & prepares JSON response  | Fetch Customer            | Check If Excel Required1               |                                                                                                                   |
| Check If Excel Required1      | If                   | Branches logic on response format        | Prepare Output Data1       | Return all data for create binary file (true) / Respond with JSON1 (false) |                                                                                                                   |
| Return all data for create binary file | Code          | Passes data for Excel conversion          | Check If Excel Required1   | Convert to Excel1                     |                                                                                                                   |
| Convert to Excel1             | ConvertToFile        | Converts JSON to Excel file               | Return all data for create binary file | Respond with File1                   |                                                                                                                   |
| Respond with File1            | RespondToWebhook     | Sends Excel file response                  | Convert to Excel1          | None                                   |                                                                                                                   |
| Respond with JSON1            | RespondToWebhook     | Sends JSON response                        | Check If Excel Required1   | None                                   |                                                                                                                   |
| Overview Note1                | Sticky Note          | Explains workflow purpose and setup      | None                      | None                                   | ## How it works ... (full content in node details section)                                                       |
| Overview Note2                | Sticky Note          | Explains request validation logic        | None                      | None                                   | ## Request & Validation ...                                                                                       |
| Overview Note3                | Sticky Note          | Explains Odoo query logic                 | None                      | None                                   | ## Search Records from Odoo ...                                                                                    |
| Overview Note4                | Sticky Note          | Explains data preparation                  | None                      | None                                   | ## Prepare data and manage response json object.                                                                  |
| Overview Note5                | Sticky Note          | Explains response format checking         | None                      | None                                   | ## Check response_format ...                                                                                       |
| Overview Note6                | Sticky Note          | Explains data preparation for binary file | None                      | None                                   | ## Prepare Data for Binary file.                                                                                   |
| Overview Note7                | Sticky Note          | Explains Excel file creation               | None                      | None                                   | ## Create excel file                                                                                               |
| Overview Note8                | Sticky Note          | Explains webhook response with Excel file | None                      | None                                   | ## Response to web-hook with excel file.                                                                           |
| Overview Note9                | Sticky Note          | Explains webhook response with JSON object | None                      | None                                   | ## Response to web-hook with json object.                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Receive Company Request1`  
   - Type: Webhook  
   - Path: `/api/v1/get-customers`  
   - Response Mode: `responseNode`  
   - Method: `GET` (default)  
   - No credentials required here.

2. **Create Function Node for Filter Preparation**  
   - Name: `Prepare Dynamic Filter1`  
   - Type: Function  
   - Code:  
     ```javascript
     const query = $json["query"] || {};
     if (!query.name || query.name.trim() === "") {
       return [{ json: { success: false, message: "Missing required parameter: name" } }];
     }
     const filters = [];
     filters.push(["name", "Like", query.name]);
     const response_format = query.response_format || "json";
     return [{ json: { filters, response_format } }];
     ```  
   - Connect output of webhook node to this function.

3. **Create Odoo Node to Fetch Customers**  
   - Name: `Fetch Customer`  
   - Type: Odoo  
   - Credentials: Configure and select your Odoo API credentials (OAuth2 or API key).  
   - Resource: `custom`  
   - Operation: `getAll`  
   - Custom Resource: `res.partner`  
   - Return All: true  
   - Fields List: `display_name`, `name`, `email`, `phone`, `mobile`, `parent_id`, `country_code`, `country_id`  
   - Filter: Use the filter expression from previous node: `[["name", "like", <name from filter>]]`  
   - Connect output of `Prepare Dynamic Filter1` to this node.

4. **Create Function Node to Prepare Output Data**  
   - Name: `Prepare Output Data1`  
   - Type: Function  
   - Code:  
     ```javascript
     if (items.length === 0 || Object.keys(items[0].json).length === 0) {
       return [{ json: { success: false, message: 'No matching company records found' } }];
     }
     const data = items.map(item => ({ ...item.json, report_generated_on: new Date().toISOString() }));
     return data.map(d => ({ json: d }));
     ```  
   - Connect output of `Fetch Customer` to this node.

5. **Create If Node to Check Response Format**  
   - Name: `Check If Excel Required1`  
   - Type: If  
   - Condition: String equals `={{ $('Prepare Dynamic Filter1').item.json.response_format }}` to `"excel"`  
   - Connect output of `Prepare Output Data1` to this node.

6. **Create Code Node to Pass All Data for Binary Creation**  
   - Name: `Return all data for create binary file`  
   - Type: Code  
   - Code: `return $input.all();`  
   - Connect the **true** output of the If node to this node.

7. **Create ConvertToFile Node to Generate Excel**  
   - Name: `Convert to Excel1`  
   - Type: ConvertToFile  
   - Operation: `xlsx`  
   - Connect output of the code node to this node.

8. **Create RespondToWebhook Node for Excel**  
   - Name: `Respond with File1`  
   - Type: RespondToWebhook  
   - Respond With: `binary`  
   - Connect output of ConvertToFile node to this node.

9. **Create RespondToWebhook Node for JSON**  
   - Name: `Respond with JSON1`  
   - Type: RespondToWebhook  
   - Respond With: `allIncomingItems`  
   - Connect the **false** output of the If node to this node.

10. **Add Sticky Notes (Optional but Recommended)**  
    - Add descriptive sticky notes explaining each block's purpose as per the overview notes content for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                      | Context or Link                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| This workflow exposes an API endpoint that returns customer data from Odoo, supporting JSON or Excel formats depending on the query parameter.                                                                                                                                                 | Overview Note1 content in workflow.                                        |
| The `name` parameter is mandatory and used for a case-sensitive partial match filter on customer names.                                                                                                                                                                                         | Overview Note2 and Note3.                                                  |
| Customize the Odoo node's `fieldsList` to include additional fields such as city or address as needed based on your Odoo schema.                                                                                                                                                                | Overview Note3.                                                            |
| For testing, use tools like Postman or a browser with URLs such as `/api/v1/get-customers?name=Demo&response_format=json` or `/api/v1/get-customers?name=Demo&response_format=excel`.                                                                                                              | Overview Note1 instructions.                                               |
| The workflow is designed to handle empty result sets gracefully by returning a JSON error message.                                                                                                                                                                                              | Covered in Prepare Output Data1 node.                                      |
| Excel conversion uses the built-in n8n `ConvertToFile` node with default settings, which can be customized if needed (e.g., sheet names, styles).                                                                                                                                                 | Overview Note7.                                                            |
| All webhook responses are synchronous; ensure network and Odoo API availability for best performance.                                                                                                                                                                                            | General best practice.                                                     |
| Odoo API credentials must be created and configured in n8n credentials manager before using the Odoo node.                                                                                                                                                                                      | Credential requirement.                                                    |

---

**Disclaimer:** The text provided is derived exclusively from an automated n8n workflow. The process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.