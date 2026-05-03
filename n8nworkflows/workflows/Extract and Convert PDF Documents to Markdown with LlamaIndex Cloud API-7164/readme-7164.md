Extract and Convert PDF Documents to Markdown with LlamaIndex Cloud API

https://n8nworkflows.xyz/workflows/extract-and-convert-pdf-documents-to-markdown-with-llamaindex-cloud-api-7164


# Extract and Convert PDF Documents to Markdown with LlamaIndex Cloud API

### 1. Workflow Overview

This workflow automates the extraction and conversion of uploaded PDF documents into Markdown format using the LlamaIndex Cloud API. It is designed to receive a PDF file via a form submission, upload it to the LlamaIndex parsing service, poll the job status until completion, and finally retrieve the extracted content in Markdown.

The logical blocks are:

- **1.1 Input Reception:** Captures PDF files via a form trigger node.
- **1.2 Document Upload:** Uploads the received PDF to LlamaIndex API for parsing.
- **1.3 Job Status Polling:** Waits and checks the processing status repeatedly until the job is successful.
- **1.4 Content Retrieval:** Upon successful job completion, fetches the parsed Markdown content from the API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures user input from a form submission. The form allows uploading PDF files which trigger the workflow start.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: *Form Trigger*  
    - Role: Entry point, receives a PDF file uploaded by the user.  
    - Configuration: Form titled "Upload file," with a single file input field accepting PDFs only (`acceptFileTypes: pdf`).  
    - Inputs: HTTP webhook triggered by form submission.  
    - Outputs: Passes uploaded file data to the next node.  
    - Edge cases: File size limits or unsupported formats might cause upload failure. Webhook must be accessible publicly.  
    - Version: 2.2

#### 1.2 Document Upload

- **Overview:**  
  This block uploads the received PDF file to the LlamaIndex Cloud API for parsing.

- **Nodes Involved:**  
  - Upload_doc

- **Node Details:**  
  - **Upload_doc**  
    - Type: *HTTP Request*  
    - Role: Sends a POST request with the PDF as multipart/form-data to LlamaIndex `/parsing/upload` endpoint.  
    - Configuration:  
      - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`  
      - Method: POST  
      - Headers: Accepts JSON, Content-Type multipart/form-data  
      - Body: Multipart form-data with `file` containing the uploaded binary PDF from the previous node.  
      - Authentication: HTTP Bearer token (credentials named "Bearer Auth account" and "LlamaCloud")  
    - Inputs: Receives binary file data from the form submission node.  
    - Outputs: Returns JSON with an upload job ID for status tracking.  
    - Edge cases: Auth failures, network errors, file size or format rejections by API.  
    - Version: 4.2

#### 1.3 Job Status Polling

- **Overview:**  
  This block waits and periodically checks the status of the submitted parsing job until it completes successfully.

- **Nodes Involved:**  
  - Wait  
  - Status Verification  
  - If  
  - Wait2

- **Node Details:**  
  - **Wait**  
    - Type: *Wait*  
    - Role: Pauses the workflow for 30 seconds before checking job status.  
    - Configuration: Wait time of 30 seconds.  
    - Inputs: Triggered after document upload.  
    - Outputs: Proceeds to status verification.  
    - Edge cases: Fixed delay might cause inefficiency if job completes faster or slower.  
    - Version: 1.1

  - **Status Verification**  
    - Type: *HTTP Request*  
    - Role: Queries LlamaIndex API for current job status using job ID from upload response.  
    - Configuration:  
      - URL: `https://api.cloud.llamaindex.ai/api/parsing/job/{{ $('Upload_doc').item.json.id }}` (dynamic injection of job ID)  
      - Method: GET  
      - Headers: Accept JSON  
      - Authentication: HTTP Bearer token (same as Upload_doc)  
    - Inputs: Receives trigger from Wait or Wait2.  
    - Outputs: Returns JSON with job status to the If node.  
    - Edge cases: Auth errors, invalid job ID, or network failures.  
    - Version: 4.2

  - **If**  
    - Type: *If Condition*  
    - Role: Checks if the job status equals "SUCCESS."  
    - Configuration: Condition on `$json.status == "SUCCESS"`  
    - Inputs: Receives job status JSON from Status Verification.  
    - Outputs:  
      - If true: forwards to Content Extraction node.  
      - If false: forwards to Wait2 to wait longer and retry.  
    - Edge cases: Possible unexpected status values or missing status field.  
    - Version: 2.2

  - **Wait2**  
    - Type: *Wait*  
    - Role: Additional wait of 60 seconds before re-checking job status.  
    - Configuration: 60 seconds wait time.  
    - Inputs: Activated when job is not yet successful.  
    - Outputs: Loops back to Status Verification for re-check.  
    - Edge cases: Long delays can slow overall processing.  
    - Version: 1.1

#### 1.4 Content Retrieval

- **Overview:**  
  Once the job is successful, this block fetches the extracted Markdown content from the LlamaIndex API.

- **Nodes Involved:**  
  - Content extraction

- **Node Details:**  
  - **Content extraction**  
    - Type: *HTTP Request*  
    - Role: Retrieves the Markdown result of the parsed document.  
    - Configuration:  
      - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown` (uses job ID from If node output)  
      - Method: GET  
      - Headers: Accept JSON  
      - Authentication: HTTP Bearer token (same as above)  
    - Inputs: Triggered by If node upon success.  
    - Outputs: JSON containing Markdown content of the PDF.  
    - Edge cases: Auth or API errors, missing or incomplete result data.  
    - Version: 4.2

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                    | Input Node(s)       | Output Node(s)     | Sticky Note                                      |
|--------------------|---------------------|----------------------------------|---------------------|--------------------|-------------------------------------------------|
| On form submission  | Form Trigger        | Receives PDF file input via form | -                   | Upload_doc         |                                                 |
| Upload_doc         | HTTP Request        | Uploads PDF to LlamaIndex API    | On form submission  | Wait               |                                                 |
| Wait               | Wait                | Waits 30 seconds before status check | Upload_doc         | Status Verification |                                                 |
| Status Verification | HTTP Request        | Checks parsing job status        | Wait, Wait2         | If                 |                                                 |
| If                 | If Condition        | Checks if job status is SUCCESS  | Status Verification | Content extraction, Wait2 |                                                 |
| Wait2              | Wait                | Waits 60 seconds before retrying status check | If (false branch) | Status Verification |                                                 |
| Content extraction  | HTTP Request        | Retrieves Markdown content       | If (true branch)    | -                  |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission`  
   - Type: Form Trigger (version 2.2)  
   - Parameters:  
     - Form title: "Upload file"  
     - Form fields: One file input field labeled "File" accepting only PDFs (`acceptFileTypes: pdf`)  
   - This node will start the workflow upon file upload.

2. **Add an HTTP Request node**  
   - Name: `Upload_doc`  
   - Type: HTTP Request (version 4.2)  
   - Connect input from `On form submission` node.  
   - Parameters:  
     - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`  
     - Method: POST  
     - Content type: multipart/form-data  
     - Body: multipart with one parameter named `file` mapped to the binary data from the form upload.  
     - Headers:  
       - Accept: application/json  
       - Content-Type: multipart/form-data  
     - Authentication: HTTP Bearer Authentication using valid LlamaIndex API token credentials.  
   - Credentials: Configure HTTP Bearer Auth with your LlamaIndex API token.

3. **Add a Wait node**  
   - Name: `Wait`  
   - Type: Wait (version 1.1)  
   - Connect input from `Upload_doc`.  
   - Parameters: Wait for 30 seconds.

4. **Add an HTTP Request node**  
   - Name: `Status Verification`  
   - Type: HTTP Request (version 4.2)  
   - Connect input from `Wait`.  
   - Parameters:  
     - URL: `https://api.cloud.llamaindex.ai/api/parsing/job/{{ $('Upload_doc').item.json.id }}` (use expression to inject job ID from upload response)  
     - Method: GET  
     - Headers: Accept application/json  
     - Authentication: Same HTTP Bearer Auth credentials as `Upload_doc`.

5. **Add an If node**  
   - Name: `If`  
   - Type: If Condition (version 2.2)  
   - Connect input from `Status Verification`.  
   - Parameters:  
     - Condition: Check if `$json.status` equals `"SUCCESS"` (case-sensitive).  
   - Outputs:  
     - True branch leads to `Content extraction`.  
     - False branch leads to `Wait2`.

6. **Add a Wait node**  
   - Name: `Wait2`  
   - Type: Wait (version 1.1)  
   - Connect input from `If` false branch.  
   - Parameters: Wait for 60 seconds.

7. **Connect `Wait2` output back to `Status Verification` input**  
   - This creates a polling loop until the job status becomes SUCCESS.

8. **Add an HTTP Request node**  
   - Name: `Content extraction`  
   - Type: HTTP Request (version 4.2)  
   - Connect input from `If` true branch.  
   - Parameters:  
     - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown` (inject job ID from the If node’s input JSON)  
     - Method: GET  
     - Headers: Accept application/json  
     - Authentication: Same HTTP Bearer Auth credentials.

9. **Set credentials for all HTTP Request nodes requiring LlamaIndex API authentication**  
   - Use the HTTP Bearer Auth credential with your API token.

10. **Save and activate the workflow**  
    - Test by submitting a PDF via the form to confirm the full flow from upload to Markdown extraction.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                               |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow uses LlamaIndex Cloud API to parse PDFs and extract Markdown content automatically.  | Official API docs: https://docs.llamaindex.ai/                |
| Wait nodes implement polling with fixed intervals; consider optimizing for faster job completion.  | Polling strategy in asynchronous APIs                          |
| Ensure HTTP Bearer authentication tokens are securely stored in n8n credentials.                   | n8n documentation on credentials management                    |
| Form Trigger node requires a publicly accessible webhook endpoint for form submissions.            | n8n Forms documentation                                        |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.