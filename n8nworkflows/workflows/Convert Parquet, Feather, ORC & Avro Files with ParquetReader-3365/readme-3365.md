Convert Parquet, Feather, ORC & Avro Files with ParquetReader

https://n8nworkflows.xyz/workflows/convert-parquet--feather--orc---avro-files-with-parquetreader-3365


# Convert Parquet, Feather, ORC & Avro Files with ParquetReader

### 1. Workflow Overview

This workflow enables users to upload columnar data files in Parquet, Feather, ORC, or Avro formats via a webhook and instantly receive a structured JSON preview of the file’s contents. It leverages the ParquetReader API to parse and convert these binary file formats into JSON, including detailed data rows, schema, and metadata. The workflow is ideal for validating file schemas, previewing data before ETL or transformation, automating quality assurance, and integrating file validation into broader automated pipelines.

**Logical Blocks:**

- **1.1 Input Reception:** Receives the uploaded file via an HTTP webhook configured to accept POST requests with multipart form-data.
- **1.2 API Forwarding:** Sends the received file to the ParquetReader API endpoint as multipart/form-data for parsing.
- **1.3 Response Parsing:** Processes the JSON response from the API, converting stringified JSON fields into usable JSON objects.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP POST requests containing a file upload. It captures the file in binary form under the property `file` for further processing.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger (HTTP POST)  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `convert` (endpoint URL suffix)  
      - Binary Property Name: `file` (expects the uploaded file in this form-data field)  
      - Response Mode: Returns data from the last node in the workflow as the HTTP response  
    - Input Connections: None (entry point)  
    - Output Connections: Sends data to "Send to Parquet API" node  
    - Edge Cases / Potential Failures:  
      - Missing or incorrectly named file field in form-data causes failure or empty input  
      - Large file uploads may cause timeout or memory issues depending on n8n instance limits  
      - HTTP method other than POST will not trigger the workflow  
    - Notes:  
      - Example usage provided via curl command in sticky note  
      - Can be triggered externally or reused from other n8n workflows via HTTP Request node  

#### 1.2 API Forwarding

- **Overview:**  
  This block forwards the uploaded binary file to the ParquetReader API endpoint for parsing. It sends the file as multipart/form-data under the field name `file` and expects a JSON response containing parsed data.

- **Nodes Involved:**  
  - Send to Parquet API

- **Node Details:**

  - **Send to Parquet API**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://api.parquetreader.com/parquet?source=n8n` (includes a query parameter identifying the source)  
      - HTTP Method: POST  
      - Body Content Type: multipart/form-data  
      - Binary Property Name: `file0` (the binary field passed from the webhook is renamed internally to `file0` for sending)  
      - JSON Parameters: Enabled (to parse JSON response automatically)  
      - Sends binary data directly from the webhook node  
    - Input Connections: Receives binary file from Webhook node  
    - Output Connections: Sends API response JSON to "Parse API Response" node  
    - Edge Cases / Potential Failures:  
      - Network errors or API downtime causing request failure  
      - File size or format not supported by the API (though API supports Parquet, Avro, ORC, Feather)  
      - Unexpected API response format or error messages  
      - No authentication required currently, but future API limits or auth may require credential updates  
    - Version Requirements: Compatible with n8n HTTP Request node version 1 or higher  

#### 1.3 Response Parsing

- **Overview:**  
  This block processes the API response, which contains JSON fields that may be stringified JSON themselves. It parses these string fields into proper JSON objects or arrays for easier downstream use or return.

- **Nodes Involved:**  
  - Parse API Response

- **Node Details:**

  - **Parse API Response**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Custom JavaScript code that:  
        - Checks if `data` property is a string; if so, parses it into an array  
        - Checks if `meta_data` property is a string; if so, parses it into an object  
        - Returns the modified item for output  
    - Input Connections: Receives JSON response from "Send to Parquet API" node  
    - Output Connections: None (last node; data returned to webhook caller)  
    - Edge Cases / Potential Failures:  
      - Malformed JSON in `data` or `meta_data` fields causing parse errors  
      - Missing expected properties in the API response  
      - If the API changes response structure, the code may need updating  
    - Version Requirements: Requires n8n version supporting Code node with JavaScript execution (version 2 used here)  

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role          | Input Node(s) | Output Node(s)      | Sticky Note                                                                                                  |
|---------------------|--------------------|-------------------------|---------------|---------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook Trigger    | Receives uploaded file   | None          | Send to Parquet API | Example trigger flow via curl provided. Accepts file upload as multipart/form-data with field name `file`.   |
| Send to Parquet API  | HTTP Request       | Sends file to API       | Webhook       | Parse API Response   | Sends binary file as multipart/form-data to ParquetReader API endpoint. No auth required currently.          |
| Parse API Response   | Code (JavaScript)  | Parses stringified JSON  | Send to Parquet API | None                | Parses `data` and `meta_data` fields from string to JSON objects for easier downstream use.                  |
| Sticky Note         | Sticky Note        | Documentation           | None          | None                | Contains detailed usage instructions, example curl command, and explanation of workflow purpose and usage.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set HTTP Method to POST  
   - Set Path to `convert` (or desired endpoint suffix)  
   - Enable "Binary Property Name" and set it to `file` (this is the form-data field expected)  
   - Set Response Mode to "Last Node" to return final output from the workflow  

2. **Create HTTP Request Node (Send to Parquet API)**  
   - Type: HTTP Request  
   - Connect Webhook node output to this node input  
   - Set URL to `https://api.parquetreader.com/parquet?source=n8n`  
   - Set HTTP Method to POST  
   - Under Options, set "Body Content Type" to `multipart-form-data`  
   - Enable "Send Binary Data" and set "Binary Property Name" to `file0` (this matches the binary data from webhook; n8n automatically maps the uploaded file to `file0`)  
   - Enable "JSON/RAW Parameters" to parse JSON response automatically  

3. **Create Code Node (Parse API Response)**  
   - Type: Code  
   - Connect HTTP Request node output to this node input  
   - Use JavaScript code to parse stringified JSON fields:  
     ```javascript
     const item = items[0];

     if (typeof item.json.data === 'string') {
       item.json.data = JSON.parse(item.json.data);
     }

     if (typeof item.json.meta_data === 'string') {
       item.json.meta_data = JSON.parse(item.json.meta_data);
     }

     return [item];
     ```  
   - This ensures `data` and `meta_data` are proper JSON objects/arrays for downstream use or response  

4. **Set Workflow Execution Order**  
   - Connect nodes in sequence:  
     Webhook → Send to Parquet API → Parse API Response  

5. **Activate Workflow**  
   - Save and activate the workflow to start listening for incoming file uploads  

6. **Optional: Add Sticky Note**  
   - Add a Sticky Note node with usage instructions, example curl command, and workflow purpose for documentation inside n8n  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The ParquetReader API supports .parquet, .avro, .orc, and .feather file formats without requiring authentication currently.                    | https://parquetreader.com                                                                        |
| Example curl command to trigger the webhook: `curl -X POST http://localhost:5678/webhook-test/convert -F "file=@converted.parquet"`            | Included in Sticky Note and workflow description                                                |
| The workflow returns a JSON response including `data` (rows), `schema` (column definitions), and `meta_data` (file metadata)                   | See example JSON response in workflow description                                               |
| This workflow is suitable for integration into ETL pipelines, automated QA, and schema validation workflows without custom coding               | Workflow description and use case section                                                       |
| Future API usage limits or authentication might require updating the HTTP Request node credentials or parameters                               | API Info section in workflow description                                                        |

---

This completes the detailed, structured reference documentation for the "Convert Parquet, Avro, ORC & Feather via ParquetReader to JSON" n8n workflow.