Extract and Structure Invoice Data with DocSumo and Export to Excel

https://n8nworkflows.xyz/workflows/extract-and-structure-invoice-data-with-docsumo-and-export-to-excel-6195


# Extract and Structure Invoice Data with DocSumo and Export to Excel

### 1. Workflow Overview

This workflow automates the extraction and structuring of invoice data submitted via a form, using the DocSumo OCR and data extraction API, and exports the processed invoice information into an Excel file. It is designed for use cases where users upload invoice documents that need to be parsed into structured data for accounting, auditing, or record-keeping.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Captures invoice files uploaded through a web form.
- **1.2 Document Upload and Processing:** Uploads the invoice file to DocSumo, retrieves processing results, and gets detailed parsed data.
- **1.3 Data Extraction and Transformation:** Extracts relevant invoice header and line item details from the DocSumo response, restructures it into a tabular format.
- **1.4 Export to Excel:** Converts the structured data into an Excel file for download or further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for form submissions containing invoice files and triggers the workflow upon upload.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point capturing file uploads from a form titled "file" with a single file field named "file".  
    - *Configuration:* Single file upload allowed, field labeled "file".  
    - *Expressions/Variables:* N/A  
    - *Input:* External web form submission  
    - *Output:* Passes the uploaded file to the next node for processing  
    - *Edge Cases:* No file uploaded or multiple files if form settings changed, network interruptions during upload  
    - *Version:* 2.2  

---

#### 2.2 Document Upload and Processing

- **Overview:**  
  Takes the uploaded file, sends it to DocSumo API to create a document for invoice parsing, then fetches detailed and simplified parsed data.

- **Nodes Involved:**  
  - HTTP Request  
  - HTTP Request1  
  - HTTP Request2

- **Node Details:**  

  - **HTTP Request**  
    - *Type:* HTTP Request  
    - *Role:* Uploads the invoice file to DocSumo's upload API endpoint with multipart/form-data.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://app.docsumo.com/api/v1/eevee/apikey/upload/`  
      - Body: Includes `type=invoice`, the uploaded binary file, `skip_review=true` for automated processing  
      - Authentication: HTTP header authentication via generic credential  
    - *Expressions:* Uses binary data field "file" from form submission  
    - *Input:* File from "On form submission" node  
    - *Output:* JSON response with document ID for next requests  
    - *Edge Cases:* Authentication failures, file size limits, invalid file types, API downtime  
    - *Version:* 4.2  

  - **HTTP Request1**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves detailed document information using the document ID from the upload response.  
    - *Configuration:*  
      - Method: GET (default)  
      - URL templated as `https://app.docsumo.com/api/v1/eevee/apikey/documents/detail/{{ $json.data.document[0].doc_id }}`  
      - Authentication: Same generic HTTP header-based auth  
    - *Expressions:* Uses dynamic document ID from previous node's JSON data  
    - *Input:* Output of HTTP Request node  
    - *Output:* Detailed document data JSON  
    - *Edge Cases:* Missing or invalid document ID, API failures, unexpected response structure  
    - *Version:* 4.2  

  - **HTTP Request2**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves simplified structured data of the invoice document to ease data extraction.  
    - *Configuration:*  
      - Method: GET  
      - URL templated as `https://app.docsumo.com/api/v1/eevee/apikey/data/simplified/{{ $json.data.document.doc_id }}/`  
      - Headers: `accept: application/json`  
      - Authentication: Same generic HTTP header auth  
    - *Expressions:* Uses dynamic document ID from previous node's output  
    - *Input:* Output of HTTP Request1 node  
    - *Output:* Simplified JSON data with invoice fields and line items  
    - *Edge Cases:* Authentication issues, data not yet processed, empty or malformed data  
    - *Version:* 4.2  

---

#### 2.3 Data Extraction and Transformation

- **Overview:**  
  Processes the JSON data to extract invoice header fields and line items, merging them into a flat, tabular format suitable for exporting.

- **Nodes Involved:**  
  - Code  
  - Sticky Note (instructional)

- **Node Details:**  

  - **Code**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses DocSumo's complex JSON output to extract key invoice details and line items into a structured array of objects.  
    - *Configuration:*  
      - Custom JavaScript function that:  
        - Extracts invoice header fields (invoice number, issue date, buyer/seller details, tax amounts)  
        - Iterates over line items to include all columns (item code, description, quantity, pricing, taxes, delivery date)  
        - Handles missing or undefined fields safely by defaulting to empty strings or zeros  
        - Logs errors during processing without stopping the workflow  
      - Returns an array of JSON objects representing each line item enriched with header data  
    - *Expressions:* Uses optional chaining and safe access for nested JSON data  
    - *Input:* JSON from HTTP Request2 node  
    - *Output:* Array of JSON objects, each representing a complete invoice line item with header context  
    - *Edge Cases:* Unexpected data structure, missing fields, runtime errors in JS (caught and logged)  
    - *Version:* 2  

  - **Sticky Note**  
    - *Content:* "PLease adjust Code ou can customize header or line item extraction by editing the Code node as needed."  
    - *Role:* Instruction to users for customization of data extraction logic  
    - *Position:* Near Code node  

---

#### 2.4 Export to Excel

- **Overview:**  
  Converts the processed JSON data array into an Excel spreadsheet file for download, sharing, or archival.

- **Nodes Involved:**  
  - Convert to File

- **Node Details:**  

  - **Convert to File**  
    - *Type:* Convert To File  
    - *Role:* Converts JSON data into an XLS (Excel) file format.  
    - *Configuration:*  
      - Operation: XLS export  
      - Binary property name: default (empty string, uses standard input)  
    - *Input:* JSON array from Code node  
    - *Output:* Binary Excel file attached to workflow output  
    - *Edge Cases:* Large datasets could impact performance, invalid JSON structure could break conversion  
    - *Version:* 1.1  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                       | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                 |
|---------------------|---------------------|------------------------------------|-----------------------|----------------------|---------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Capture uploaded invoice file       | (external form)        | HTTP Request          |                                                                                             |
| HTTP Request        | HTTP Request        | Upload invoice file to DocSumo API  | On form submission     | HTTP Request1         |                                                                                             |
| HTTP Request1       | HTTP Request        | Fetch detailed document info        | HTTP Request          | HTTP Request2         |                                                                                             |
| HTTP Request2       | HTTP Request        | Fetch simplified parsed invoice data| HTTP Request1         | Code                  |                                                                                             |
| Code                | Code                | Extract and restructure invoice data| HTTP Request2         | Convert to File       | PLease adjust Code ou can customize header or line item extraction by editing the Code node |
| Convert to File     | Convert To File     | Export structured data to Excel     | Code                   | (workflow output)     |                                                                                             |
| Sticky Note         | Sticky Note         | Instructional note on Code node     | N/A                    | N/A                   | PLease adjust Code ou can customize header or line item extraction by editing the Code node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and add a Form Trigger node:**  
   - Name: `On form submission`  
   - Configure form title as `"file"`  
   - Add one field: type `file`, label `"file"`, allow only one file upload  
   - This node will trigger when a user uploads an invoice file via the form

2. **Add an HTTP Request node to upload the file to DocSumo:**  
   - Name: `HTTP Request`  
   - Method: POST  
   - URL: `https://app.docsumo.com/api/v1/eevee/apikey/upload/`  
   - Authentication: Set up generic HTTP header authentication with your DocSumo API key  
   - Body Content Type: multipart-form-data  
   - Body parameters:  
     - `type` = `"invoice"`  
     - `file` = Binary data from `On form submission` node, field name `"file"`  
     - `skip_review` = `"true"` (to bypass manual review)  
   - Connect `On form submission` node output to this node

3. **Add an HTTP Request node to fetch detailed document info:**  
   - Name: `HTTP Request1`  
   - Method: GET (default)  
   - URL: Use expression `https://app.docsumo.com/api/v1/eevee/apikey/documents/detail/{{ $json.data.document[0].doc_id }}` to dynamically use document ID from previous node  
   - Authentication: Same generic HTTP header auth with DocSumo key  
   - Connect `HTTP Request` node output to this node

4. **Add an HTTP Request node to fetch simplified parsed data:**  
   - Name: `HTTP Request2`  
   - Method: GET  
   - URL: Use expression `https://app.docsumo.com/api/v1/eevee/apikey/data/simplified/{{ $json.data.document.doc_id }}/`  
   - Headers: Add header `accept` = `application/json`  
   - Authentication: Same generic HTTP header auth  
   - Connect `HTTP Request1` node output to this node

5. **Add a Code node to process and restructure the JSON data:**  
   - Name: `Code`  
   - Language: JavaScript  
   - Paste the JavaScript code that:  
     - Extracts invoice header fields (Invoice Number, Issue Date, Buyer/Seller details, Totals)  
     - Iterates over line items to flatten them with header info included  
     - Handles missing or empty fields safely  
     - Logs errors without breaking the workflow  
   - Connect `HTTP Request2` node output to this node

6. **Add a Convert to File node to export JSON to Excel:**  
   - Name: `Convert to File`  
   - Operation: `xls` (Excel)  
   - Binary Property Name: default (empty)  
   - Connect `Code` node output to this node

7. **Optionally, add a Sticky Note near the Code node:**  
   - Content: `"PLease adjust Code ou can customize header or line item extraction by editing the Code node as needed."`

8. **Credentials Setup:**  
   - Create a generic HTTP header authentication credential in n8n with the DocSumo API key. Use this credential in all HTTP Request nodes.  
   - No additional credentials are needed for other nodes.

9. **Activate and test the workflow:**  
   - Submit an invoice file via the form URL provided by the Form Trigger node.  
   - Observe the workflow execution to verify data extraction and Excel export.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The Code node is the main customization point. Adjust it to map different invoice fields or handle various invoice layouts. | Located near the "Code" node in the workflow                                                    |
| DocSumo API documentation: https://docs.docsumo.com/                                            | Official API reference for endpoints used in this workflow                                      |
| Form Trigger node provides a webhook URL that can be embedded in a web page or used in API calls | Useful for integrating file uploads directly into the workflow                                  |

---

**Disclaimer:** The content provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and includes no illegal, offensive, or protected elements. All data handled is legal and public.