Merge PDF files using ConvertAPI

https://n8nworkflows.xyz/workflows/merge-pdf-files-using-convertapi-2298


# Merge PDF files using ConvertAPI

### 1. Workflow Overview

This workflow automates the process of merging two PDF files into a single document using the ConvertAPI service. It is designed primarily for developers and organizations that need to programmatically combine PDF files without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Input Acquisition:** Initiates the workflow manually and downloads two remote PDF files.
- **1.2 PDF Merging via API:** Sends the two downloaded PDFs to ConvertAPI’s merge endpoint to combine them.
- **1.3 Output Storage:** Saves the merged PDF locally on the file system for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Acquisition

**Overview:**  
This block starts the workflow upon manual trigger and handles downloading the two source PDF files from their remote URLs.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Download first remote PDF File  
- Download second PDF File  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually for testing or on-demand execution.  
  - Configuration: No parameters; simply triggers the chain.  
  - Inputs: None  
  - Outputs: Connected to “Download first remote PDF File” node.  
  - Edge cases: None, as it depends on user interaction.  

- **Download first remote PDF File**  
  - Type: HTTP Request  
  - Role: Downloads the first PDF file from a public URL.  
  - Configuration:  
    - URL set to `https://cdn.convertapi.com/public/files/demo.pdf`.  
    - Response format: File, stored under output property `data1`.  
  - Inputs: Connected from the manual trigger node.  
  - Outputs: Connected to “Download second PDF File” node.  
  - Edge cases:  
    - Network failures or unreachable URL result in download failure.  
    - HTTP errors (404, 500) must be handled externally or cause workflow failure.  

- **Download second PDF File**  
  - Type: HTTP Request  
  - Role: Downloads the second PDF file from a public URL.  
  - Configuration:  
    - URL set to `https://cdn.convertapi.com/public/files/demo2.pdf`.  
    - Response format: File, stored under output property `data2`.  
  - Inputs: Connected from “Download first remote PDF File”.  
  - Outputs: Connected to “PDF merge API HTTP Request” node.  
  - Edge cases: Similar to the first download node, with potential network or HTTP errors.  

---

#### 2.2 PDF Merging via API

**Overview:**  
This block sends the two downloaded PDF files to the ConvertAPI merge endpoint, authenticates the request, and receives the merged PDF as a file response.

**Nodes Involved:**  
- PDF merge API HTTP Request  
- Sticky Note (informational)  

**Node Details:**  

- **PDF merge API HTTP Request**  
  - Type: HTTP Request  
  - Role: Performs the actual merging of PDF files using a POST request to ConvertAPI.  
  - Configuration:  
    - URL: `https://v2.convertapi.com/convert/pdf/to/merge` (ConvertAPI endpoint for PDF merging).  
    - Method: POST  
    - Authentication: HTTP Query Auth using a secret key stored in credentials (`Query Auth account`).  
    - Content Type: Multipart form-data for file upload.  
    - Body Parameters:  
      - `files[0]` linked to the binary data from `data1` (first PDF).  
      - `files[1]` linked to the binary data from `data2` (second PDF).  
    - Header Parameters: `Accept: application/octet-stream` to specify binary file response.  
    - Response: File output stored as `data`.  
  - Inputs: Connected from “Download second PDF File”.  
  - Outputs: Connected to “Write Result File to Disk”.  
  - Version-specific: Uses HTTP Request node v4.2 for advanced configurations such as multipart-form-data and binary data handling.  
  - Edge cases:  
    - Authentication failure if secret is invalid or missing.  
    - API rate limits or quota exceeded errors.  
    - Network timeouts or endpoint unavailability.  
    - File format issues if input files are corrupt or unsupported.  

- **Sticky Note**  
  - Content:  
    ```
    ## Authentication
    Conversion requests must be authenticated. Please create 
    [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin)
    ```  
  - Role: Provides important information about authentication requirements to users.  
  - Applies contextually to the PDF merge API node.  

---

#### 2.3 Output Storage

**Overview:**  
This block saves the merged PDF file received from the API to the local file system under the specified filename.

**Nodes Involved:**  
- Write Result File to Disk  

**Node Details:**  

- **Write Result File to Disk**  
  - Type: Read/Write File  
  - Role: Writes the merged PDF binary data to a file named `document.pdf` on disk.  
  - Configuration:  
    - Operation: Write  
    - Filename: `document.pdf` (relative to n8n’s working directory).  
    - Data property: Uses the binary data property `data1` (mapped from the previous node’s output property `data`).  
  - Inputs: Connected from “PDF merge API HTTP Request”.  
  - Outputs: None (final step).  
  - Edge cases:  
    - File system permissions issues preventing write.  
    - Disk full or quota exceeded.  
    - Invalid file path or naming issues.  

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                          |
|------------------------------|-------------------------|------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Initiates workflow manually        | —                           | Download first remote PDF File |                                                                                                    |
| Download first remote PDF File| HTTP Request            | Downloads first PDF from URL       | When clicking ‘Test workflow’ | Download second PDF File      |                                                                                                    |
| Download second PDF File      | HTTP Request            | Downloads second PDF from URL      | Download first remote PDF File | PDF merge API HTTP Request    |                                                                                                    |
| PDF merge API HTTP Request    | HTTP Request            | Merges two PDFs via ConvertAPI     | Download second PDF File      | Write Result File to Disk     | ## Authentication: Conversion requests must be authenticated. Please create [ConvertAPI account](https://www.convertapi.com/a/signin) |
| Write Result File to Disk     | Read/Write File         | Saves merged PDF locally            | PDF merge API HTTP Request    | —                           |                                                                                                    |
| Sticky Note                  | Sticky Note             | Provides authentication info       | —                           | —                           | ## Authentication: Conversion requests must be authenticated. Please create [ConvertAPI account](https://www.convertapi.com/a/signin) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger (`n8n-nodes-base.manualTrigger`)  
   - No parameters required.  
   - This will serve as the workflow's starting point.  

2. **Create HTTP Request Node - Download first PDF**  
   - Node Type: HTTP Request (`n8n-nodes-base.httpRequest`)  
   - Parameters:  
     - URL: `https://cdn.convertapi.com/public/files/demo.pdf`  
     - Response Format: File (set under Options → Response → Response Format)  
     - Output Property Name: `data1`  
   - Connect manual trigger node output to this node input.  

3. **Create HTTP Request Node - Download second PDF**  
   - Node Type: HTTP Request (`n8n-nodes-base.httpRequest`)  
   - Parameters:  
     - URL: `https://cdn.convertapi.com/public/files/demo2.pdf`  
     - Response Format: File  
     - Output Property Name: `data2`  
   - Connect output of the first download node to this node.  

4. **Create HTTP Request Node - PDF merge API call**  
   - Node Type: HTTP Request (`n8n-nodes-base.httpRequest`)  
   - Parameters:  
     - URL: `https://v2.convertapi.com/convert/pdf/to/merge`  
     - Method: POST  
     - Content Type: multipart-form-data  
     - Authentication: Set to HTTP Query Auth with credentials (see next step).  
     - Header Parameters: Add header with key `Accept` and value `application/octet-stream`  
     - Body Parameters (multipart form-data):  
       - `files[0]`: formBinaryData type, linked to `data1` (first PDF binary data)  
       - `files[1]`: formBinaryData type, linked to `data2` (second PDF binary data)  
     - Response Format: File, output property name `data`  
   - Connect output of second PDF download node to this node.  

5. **Set up Credentials for HTTP Query Auth**  
   - Create new Credential of type HTTP Query Auth in n8n.  
   - Set the query parameter key to `secret` (or as required by ConvertAPI).  
   - Use your ConvertAPI authentication secret as the value.  
   - Assign these credentials to the “PDF merge API HTTP Request” node.  

6. **Create Read/Write File Node - Save merged PDF**  
   - Node Type: Read/Write File (`n8n-nodes-base.readWriteFile`)  
   - Parameters:  
     - Operation: Write  
     - File Name: `document.pdf`  
     - Data Property Name: `=data1` (maps to the output file from previous node)  
   - Connect output of "PDF merge API HTTP Request" node to this node.  

7. **Add Sticky Note (Optional)**  
   - To provide users with authentication info and ConvertAPI link.  

8. **Validate and Test Workflow**  
   - Trigger manually via the manual trigger node.  
   - Check the file named `document.pdf` is created with merged content.  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Conversion requests must be authenticated. Create a ConvertAPI account to get an authentication secret. | [ConvertAPI Sign-in](https://www.convertapi.com/a/signin)                                          |
| All ConvertAPI endpoints can be explored here for customization.                                   | [ConvertAPI API Documentation](https://www.convertapi.com/api)                                     |
| The workflow uses multipart form-data to send binary files to the API, which requires HTTP Request node v4.2 or newer. | n8n HTTP Request node version requirement                                                           |

---

This documentation provides a complete understanding, node-by-node details, and precise instructions for recreating and customizing the PDF merging workflow using n8n and ConvertAPI.