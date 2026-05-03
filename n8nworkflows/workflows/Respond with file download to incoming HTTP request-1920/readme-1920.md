Respond with file download to incoming HTTP request

https://n8nworkflows.xyz/workflows/respond-with-file-download-to-incoming-http-request-1920


# Respond with file download to incoming HTTP request

### 1. Workflow Overview

This workflow enables an end user’s browser to download a PDF file directly upon making an HTTP GET request to a specified webhook URL. It demonstrates how to serve a binary file with proper HTTP headers to trigger the browser’s "Save as" dialog without navigating away from the current page.

The workflow is organized into three logical blocks:

- **1.1 Input Reception:** Receives the incoming HTTP GET request via a webhook.
- **1.2 File Retrieval:** Fetches the PDF file from an external URL.
- **1.3 Response with Attachment:** Returns the fetched file to the client with HTTP headers configured to prompt a file download, including a dynamic filename with the current date.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block waits for an HTTP GET request at a defined endpoint, initiating the workflow execution.

- **Nodes Involved:**  
  - *On GET request*

- **Node Details:**  
  - **Node:** On GET request  
    - **Type:** Webhook  
    - **Role:** Entry point that listens for HTTP GET requests on path `/download-pdf`.  
    - **Configuration:**  
      - HTTP Method: GET  
      - Path: `download-pdf`  
      - Response Mode: `responseNode` (delegates response handling to a downstream node)  
      - No authentication or additional parameters configured.  
    - **Inputs:** None (trigger node)  
    - **Outputs:** Sends data to "Fetch binary file" node.  
    - **Version Requirements:** n8n version supporting responseMode `responseNode` in webhook nodes.  
    - **Potential Failures:**  
      - Webhook path conflicts if another workflow uses the same endpoint.  
      - Network issues preventing webhook triggering.  
      - Misconfiguration leading to no response if downstream nodes fail.

#### 1.2 File Retrieval

- **Overview:**  
  This block fetches the PDF file from a remote HTTP endpoint, preparing it for delivery.

- **Nodes Involved:**  
  - *Fetch binary file*

- **Node Details:**  
  - **Node:** Fetch binary file  
    - **Type:** HTTP Request  
    - **Role:** Downloads the binary PDF file from the external URL.  
    - **Configuration:**  
      - HTTP Method: GET (default)  
      - URL: `https://www.deutschebahn.com/resource/blob/8813300/bdf106f07186f66e4448f95aca02bd4a/Faktenblatt-ICE-L_Mai23-data.pdf`  
      - Response Format: file (binary)  
      - No authentication or headers specified.  
    - **Inputs:** Receives trigger from "On GET request" node.  
    - **Outputs:** Passes binary data to "Respond with attachment" node.  
    - **Version Requirements:** n8n version 0.148+ recommended for stable binary response support.  
    - **Potential Failures:**  
      - HTTP errors (404, 500, etc.) if the URL is unreachable or resource removed.  
      - Timeout or network failures.  
      - Large file size could cause memory issues if too big.  
      - No retries configured, so transient errors may fail the workflow.

#### 1.3 Response with Attachment

- **Overview:**  
  This block sends the binary file back to the HTTP client, setting HTTP headers to trigger a file download with a dynamic filename.

- **Nodes Involved:**  
  - *Respond with attachment*

- **Node Details:**  
  - **Node:** Respond with attachment  
    - **Type:** Respond to Webhook  
    - **Role:** Sends HTTP response including the binary PDF file with Content-Disposition header to force browser download.  
    - **Configuration:**  
      - Respond With: binary data  
      - Response Headers:  
        - `content-disposition`: `attachment; filename="my_document_{{ $now.toFormat('yyyy-MM-dd') }}.pdf"`  
          This dynamically sets the filename to include the current date (e.g., `my_document_2024-06-12.pdf`).  
    - **Inputs:** Receives binary file from "Fetch binary file".  
    - **Outputs:** None (end of workflow).  
    - **Version Requirements:** Support for expressions in header values (`$now.toFormat()`), available in recent n8n versions (v0.150+ recommended).  
    - **Potential Failures:**  
      - Expression evaluation failures if `$now` or `.toFormat()` is unavailable or misused.  
      - Missing binary data input leads to empty response or error.  
      - Response header conflicts if other headers are set elsewhere.  
      - Browser compatibility: some older browsers may ignore `content-disposition`.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role               | Input Node(s)     | Output Node(s)        | Sticky Note                                                                                   |
|-----------------------|---------------------|------------------------------|-------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| On GET request        | Webhook             | Receives HTTP GET request     | -                 | Fetch binary file      | Demonstrates how to get an end user's browser to download a file using Content-Disposition header. |
| Fetch binary file     | HTTP Request        | Downloads binary PDF file     | On GET request    | Respond with attachment |                                                                                               |
| Respond with attachment | Respond to Webhook | Sends file with download headers | Fetch binary file | -                     | Uses Content-Disposition header to set filename and trigger browser download without navigation. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a new node of type **Webhook**.  
   - Set HTTP Method to **GET**.  
   - Set the path to `download-pdf`.  
   - Set **Response Mode** to `responseNode` to delegate response handling downstream.  
   - No authentication or additional parameters required.  

2. **Create HTTP Request Node**  
   - Add a new node of type **HTTP Request**.  
   - Connect the output of the Webhook node to this node.  
   - Set the HTTP method to **GET** (default).  
   - Enter URL: `https://www.deutschebahn.com/resource/blob/8813300/bdf106f07186f66e4448f95aca02bd4a/Faktenblatt-ICE-L_Mai23-data.pdf`  
   - Under Options -> Response, set **Response Format** to `file` to fetch the binary data.  
   - Leave authentication off unless the source requires it.

3. **Create Respond to Webhook Node**  
   - Add a new node of type **Respond to Webhook**.  
   - Connect the output of the HTTP Request node to this node.  
   - Set **Respond With** to `binary`.  
   - Add a response header with:  
     - Name: `content-disposition`  
     - Value: `attachment; filename="my_document_{{ $now.toFormat('yyyy-MM-dd') }}.pdf"`  
       This uses an expression to insert the current date dynamically.  
   - No body is manually set; the binary data from the HTTP request node is automatically used.

4. **Save and Activate the Workflow**  
   - Ensure the workflow is active for the webhook to listen for requests.  
   - Test by visiting `https://<your-n8n-instance>/webhook/download-pdf` in a browser.  
   - The browser should prompt to download a PDF named like `my_document_YYYY-MM-DD.pdf`.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| The workflow uses the Content-Disposition HTTP header to force the browser to download the file.    | https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Disposition                                        |
| Original idea shared by @dickhoning on n8n Community forum for exporting Excel files via webhook.  | https://community.n8n.io/t/webhook-to-excel-file/11059/24?u=mutedjam                                                 |
| This method allows file downloads without redirecting or replacing the current browser page.        | Useful for exporting reports or documents after form submissions or application processes.                          |

---

This documentation fully covers the workflow structure, logic, node configurations, and reproduction steps to enable users or AI agents to understand, modify, or rebuild the workflow efficiently.