Generate n8n Workflow Inventory Reports with JSON & Excel Export Options

https://n8nworkflows.xyz/workflows/generate-n8n-workflow-inventory-reports-with-json---excel-export-options-10852


# Generate n8n Workflow Inventory Reports with JSON & Excel Export Options

### 1. Workflow Overview

This n8n workflow is designed to generate inventory reports of n8n workflows in a user-requested format, supporting both JSON and Excel (file) export options. It is triggered via a webhook, processes query parameters to filter workflows by status, extracts relevant metadata, and then outputs the report either as a direct JSON response or as a downloadable Excel file.

The workflow logically divides into these blocks:

- **1.1 Input Reception:** Receives incoming HTTP requests and parses query parameters.
- **1.2 Data Retrieval:** Fetches all existing workflows from the n8n instance.
- **1.3 Data Filtering and Metadata Extraction:** Filters workflows based on status and extracts relevant metadata for reporting.
- **1.4 Output Determination and Formatting:** Decides output format (JSON or Excel), formats data accordingly, and sends the response back to the requester.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives incoming HTTP webhook requests and parses any query parameters provided by the client, which may specify filters and output format preferences.

**Nodes Involved:**  
- Receive Request  
- Parse Query Params

**Node Details:**

- **Receive Request**  
  - Type: Webhook  
  - Role: Entry point to receive HTTP requests triggering the workflow.  
  - Configuration: Uses an auto-generated webhook URL; no additional parameters configured.  
  - Inputs: External HTTP request.  
  - Outputs: Passes data to Parse Query Params node.  
  - Edge Cases: Missing or malformed query parameters may affect downstream logic.

- **Parse Query Params**  
  - Type: Code (JavaScript)  
  - Role: Parses and interprets query parameters, e.g., filtering criteria and output format.  
  - Configuration: Custom JavaScript code extracts query parameters from the incoming webhook data.  
  - Expressions/Variables: Reads from `$node["Receive Request"].json` or similar to parse parameters.  
  - Inputs: Output of Receive Request.  
  - Outputs: Passes parsed parameters to Fetch All Workflows node.  
  - Edge Cases: Unexpected or invalid query parameters can cause errors or unintended filtering.

---

#### 1.2 Data Retrieval

**Overview:**  
Fetches the complete list of workflows configured in the n8n instance to provide the dataset for reporting.

**Nodes Involved:**  
- Fetch All Workflows

**Node Details:**

- **Fetch All Workflows**  
  - Type: n8n API node (n8n)  
  - Role: Calls the n8n internal API to retrieve all workflows.  
  - Configuration: Default settings to fetch all workflows; no filtering at this node.  
  - Inputs: Parsed query parameters from the previous block.  
  - Outputs: Passes workflows data to Filter by Status node.  
  - Edge Cases: API authentication issues, empty workflow list, or API timeouts.

---

#### 1.3 Data Filtering and Metadata Extraction

**Overview:**  
Filters the retrieved workflows based on status (active, inactive, etc.) as requested, then extracts relevant metadata fields (such as name, ID, creation date) to prepare for reporting.

**Nodes Involved:**  
- Filter by Status  
- Extract Workflow Metadata

**Node Details:**

- **Filter by Status**  
  - Type: Code (JavaScript)  
  - Role: Filters the full workflows list to include only those matching the requested status.  
  - Configuration: Custom JavaScript filtering logic using parsed query parameters.  
  - Inputs: Workflows list from Fetch All Workflows.  
  - Outputs: Filtered workflows list to Extract Workflow Metadata node.  
  - Edge Cases: No workflows matching status, invalid status value.

- **Extract Workflow Metadata**  
  - Type: Code (JavaScript)  
  - Role: Extracts and transforms workflow data into a structured format for reporting.  
  - Configuration: Custom code selecting fields like workflow name, ID, status, creation date.  
  - Inputs: Filtered workflows list.  
  - Outputs: Structured metadata sent to Switch Output Type node.  
  - Edge Cases: Missing fields in workflow data; code errors in extraction logic.

---

#### 1.4 Output Determination and Formatting

**Overview:**  
Determines the output format requested (JSON or Excel), formats the data accordingly, and sends the response back to the client, either directly as JSON or as a downloadable Excel file.

**Nodes Involved:**  
- Switch Output Type  
- Return JSON Response1  
- Respond to Webhook2  
- Return JSON Response  
- Convert to File  
- Respond to Webhook1

**Node Details:**

- **Switch Output Type**  
  - Type: Switch  
  - Role: Routes workflow execution based on requested output type (e.g., "json" or "excel").  
  - Configuration: Branch conditions based on parsed query parameter for output format.  
  - Inputs: Workflow metadata.  
  - Outputs:  
    - JSON Response branch to Return JSON Response1 node.  
    - Excel Response branch to Return JSON Response node.  
  - Edge Cases: Unsupported output format values.

- **Return JSON Response1**  
  - Type: Code (JavaScript)  
  - Role: Prepares the workflow metadata as a JSON response payload.  
  - Configuration: Serializes data to JSON for response.  
  - Inputs: Metadata from Switch Output Type (JSON branch).  
  - Outputs: Sends data to Respond to Webhook2 node.  

- **Respond to Webhook2**  
  - Type: Respond to Webhook  
  - Role: Sends the JSON response back to the requester.  
  - Inputs: JSON data from Return JSON Response1.  
  - Outputs: Terminates workflow.

- **Return JSON Response**  
  - Type: Code (JavaScript)  
  - Role: Prepares the metadata for Excel export by formatting data for conversion.  
  - Inputs: Metadata from Switch Output Type (Excel branch).  
  - Outputs: Passes data to Convert to File node.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts JSON data into an Excel file format.  
  - Configuration: Defaults to Excel; converts structured metadata into .xlsx file.  
  - Inputs: Data from Return JSON Response.  
  - Outputs: Passes file binary data to Respond to Webhook1.

- **Respond to Webhook1**  
  - Type: Respond to Webhook  
  - Role: Sends the Excel file as a downloadable response to the requester.  
  - Inputs: Excel file binary data.  
  - Outputs: Terminates workflow.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                             | Input Node(s)          | Output Node(s)               | Sticky Note          |
|-----------------------|-------------------------|---------------------------------------------|------------------------|------------------------------|----------------------|
| Sticky Note           | Sticky Note             | Informational/annotation                     |                        |                              |                      |
| Receive Request        | Webhook                 | Entry point, receives HTTP requests          |                        | Parse Query Params            |                      |
| Parse Query Params     | Code                    | Parses incoming query parameters              | Receive Request         | Fetch All Workflows           |                      |
| Fetch All Workflows    | n8n API node            | Retrieves all workflows from n8n instance    | Parse Query Params      | Filter by Status              |                      |
| Filter by Status       | Code                    | Filters workflows by status                   | Fetch All Workflows     | Extract Workflow Metadata     |                      |
| Extract Workflow Metadata | Code                 | Extracts relevant metadata from workflows     | Filter by Status        | Switch Output Type            |                      |
| Switch Output Type     | Switch                  | Routes flow based on output format requested | Extract Workflow Metadata | Return JSON Response1, Return JSON Response |                      |
| Return JSON Response1  | Code                    | Prepares JSON response payload                | Switch Output Type      | Respond to Webhook2           |                      |
| Respond to Webhook2    | Respond to Webhook      | Sends JSON response to requester              | Return JSON Response1   |                              |                      |
| Return JSON Response   | Code                    | Prepares data for Excel file conversion       | Switch Output Type      | Convert to File               |                      |
| Convert to File        | Convert to File         | Converts data into Excel file                  | Return JSON Response    | Respond to Webhook1           |                      |
| Respond to Webhook1    | Respond to Webhook      | Sends Excel file response to requester        | Convert to File         |                              |                      |
| Sticky Note5          | Sticky Note             | Informational/annotation                       |                        |                              |                      |
| Sticky Note7          | Sticky Note             | Informational/annotation                       |                        |                              |                      |
| Sticky Note8          | Sticky Note             | Informational/annotation                       |                        |                              |                      |
| Sticky Note10         | Sticky Note             | Informational/annotation                       |                        |                              |                      |
| Sticky Note11         | Sticky Note             | Informational/annotation                       |                        |                              |                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Receive Request`  
   - Type: Webhook  
   - Configuration: Create a new webhook with auto-generated URL, method set to GET or POST as needed. No special headers or query parameters required at this stage.

2. **Add Code Node to Parse Query Parameters**  
   - Name: `Parse Query Params`  
   - Type: Code  
   - Configuration: Write JavaScript code to extract and validate query parameters from the incoming webhook request (e.g., `status` filter, `output` format).  
   - Connect output of `Receive Request` to this node.

3. **Add n8n API Node to Fetch All Workflows**  
   - Name: `Fetch All Workflows`  
   - Type: n8n (internal API)  
   - Configuration: Use action to list all workflows from the n8n instance. No filters applied here.  
   - Connect output of `Parse Query Params` to this node.

4. **Add Code Node to Filter Workflows by Status**  
   - Name: `Filter by Status`  
   - Type: Code  
   - Configuration: JavaScript code to filter workflows based on the `status` query parameter parsed earlier.  
   - Input: Workflows list from `Fetch All Workflows`.  
   - Output: Filtered workflows array.  
   - Connect `Fetch All Workflows` output to this node.

5. **Add Code Node to Extract Workflow Metadata**  
   - Name: `Extract Workflow Metadata`  
   - Type: Code  
   - Configuration: JavaScript code to map filtered workflows to extract metadata fields such as workflow ID, name, status, created date, updated date, etc.  
   - Connect `Filter by Status` output to this node.

6. **Add Switch Node to Determine Output Type**  
   - Name: `Switch Output Type`  
   - Type: Switch  
   - Configuration: Create conditions based on the output format query parameter (e.g., if `output` equals `"json"` route to JSON branch, else route to Excel branch).  
   - Connect `Extract Workflow Metadata` output to this node.

7. **Create JSON Response Branch**  
   - Add Code Node: `Return JSON Response1`  
     - Role: Format data as JSON response.  
     - Connect `Switch Output Type` JSON branch to this node.

   - Add Respond to Webhook Node: `Respond to Webhook2`  
     - Role: Send JSON payload as HTTP response.  
     - Connect `Return JSON Response1` output to this node.

8. **Create Excel Response Branch**  
   - Add Code Node: `Return JSON Response`  
     - Role: Prepare data for Excel conversion (e.g., ensure data structure suitable for file conversion).  
     - Connect `Switch Output Type` Excel branch to this node.

   - Add Convert to File Node: `Convert to File`  
     - Role: Convert JSON data to Excel file (XLSX).  
     - Configuration: Default to Excel format with appropriate file name.  
     - Connect `Return JSON Response` output to this node.

   - Add Respond to Webhook Node: `Respond to Webhook1`  
     - Role: Send Excel binary file as HTTP response with correct content-type and disposition headers for file download.  
     - Connect `Convert to File` output to this node.

9. **Test the Workflow**  
   - Deploy and activate the webhook URL.  
   - Test with different query parameters, e.g., `?status=active&output=json` and `?status=all&output=excel`.  
   - Validate that JSON and Excel outputs are correct and that filtering works as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                      |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Useful for administrators to audit and inventory workflows with flexible output formats.              | Workflow purpose                                     |
| Excel export facilitates offline analysis and sharing of workflow metadata.                          | Output option                                        |
| Webhook trigger allows integration with external systems or manual HTTP calls.                       | Input method                                         |
| This workflow requires n8n API credentials with permissions to list workflows.                        | Credential requirement                               |
| The Convert to File node defaults to Excel, but can be adapted for CSV or other formats.             | Data export flexibility                              |

---

**Disclaimer:** The provided text derives exclusively from an n8n workflow automation. It strictly complies with content policies and contains no illegal or offensive elements. All data handled is legal and public.