Extract Specific Pages from PDFs with CustomJS API

https://n8nworkflows.xyz/workflows/extract-specific-pages-from-pdfs-with-customjs-api-3872


# Extract Specific Pages from PDFs with CustomJS API

### 1. Workflow Overview

This workflow is designed to extract specific pages from PDF files by leveraging the CustomJS PDF Toolkit API within n8n. It is ideal for users who need to selectively retrieve pages from existing PDFs without manually downloading and editing the files. The workflow is structured into three main logical blocks:

- **1.1 Input Reception:** Initiates workflow execution via a manual trigger.
- **1.2 PDF Download:** Downloads the target PDF file from a specified URL using an HTTP Request node.
- **1.3 PDF Page Extraction:** Extracts selected pages from the downloaded PDF using the CustomJS API.

This modular approach allows easy replacement of the input trigger or export method and ensures smooth integration with the CustomJS PDF extraction service.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow manually, enabling user control over when the workflow runs.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  

  - **Node Name:** When clicking ‘Test workflow’  
  - **Type:** Manual Trigger  
  - **Role:** Initiates workflow execution on user action.  
  - **Configuration:** No parameters; waits for manual execution.  
  - **Expressions/Variables:** None  
  - **Connections:** Outputs to “HTTP Request” node.  
  - **Version Requirements:** n8n base node, no special requirements.  
  - **Edge Cases:** If not triggered, workflow remains idle. Manual interaction is required to start.

#### 1.2 PDF Download

- **Overview:**  
  Downloads the PDF file from a given URL to provide the base file for page extraction.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  

  - **Node Name:** HTTP Request  
  - **Type:** HTTP Request  
  - **Role:** Fetches the PDF file from a web URL.  
  - **Configuration:**  
    - URL set to `https://www.sldttc.org/allpdf/21583473018.pdf` (example PDF file)  
    - Default HTTP method GET  
    - No authentication or advanced options configured.  
  - **Expressions/Variables:** None used; static URL.  
  - **Connections:** Receives input from “When clicking ‘Test workflow’” and outputs to “Extract Pages From PDF1”.  
  - **Version Requirements:** Version 4.2 of HTTP Request node used.  
  - **Edge Cases:**  
    - HTTP errors (404, 500, timeout) could cause failure.  
    - Large file size might cause timeout or memory issues.  
    - URL hardcoded; not dynamic in this workflow.  
  - **Sub-workflow:** None.

#### 1.3 PDF Page Extraction

- **Overview:**  
  Uses the CustomJS PDF Toolkit node to extract specified page ranges from the downloaded PDF binary data.

- **Nodes Involved:**  
  - Extract Pages From PDF1

- **Node Details:**  

  - **Node Name:** Extract Pages From PDF1  
  - **Type:** CustomJS PDF Toolkit - ExtractPages node  
  - **Role:** Extracts pages 2 to 3 from the PDF data received from the HTTP Request.  
  - **Configuration:**  
    - `pageRange` set to “2-3” (pages to extract)  
    - `field_name` set via expression `=data` (indicates the input data field containing the PDF binary)  
    - Credentials: Uses CustomJS API key stored in n8n credentials named "CustomJS account"  
  - **Expressions/Variables:**  
    - `field_name` dynamically set to `data` (the field from previous node containing PDF data)  
  - **Connections:** Input from “HTTP Request” node output; no further downstream connections.  
  - **Version Requirements:** Requires installation of `@custom-js/n8n-nodes-pdf-toolkit` community node, compatible with self-hosted n8n environments only.  
  - **Edge Cases:**  
    - API key authentication failure.  
    - Invalid page range syntax or pages out of bounds.  
    - Network or API timeout errors.  
    - PDF data format errors (corrupted or unsupported PDFs).  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                   | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                  |
|---------------------------|---------------------------------------|---------------------------------|-------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                        | Starts workflow on user command  | None                          | HTTP Request                 |                                                                                                              |
| HTTP Request              | HTTP Request                          | Downloads PDF from URL            | When clicking ‘Test workflow’  | Extract Pages From PDF1       |                                                                                                              |
| Extract Pages From PDF1   | CustomJS PDF Toolkit - ExtractPages  | Extracts pages 2-3 from PDF      | HTTP Request                  | None                         | Community nodes can only be installed on self-hosted instances of n8n. Use CustomJS API key for extraction. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node. No parameters needed. This node starts the workflow by user interaction.

2. **Add HTTP Request Node:**  
   - Add an HTTP Request node.  
   - Set the URL parameter to the desired PDF file URL (e.g., `https://www.sldttc.org/allpdf/21583473018.pdf`).  
   - Leave HTTP method as GET.  
   - Connect the output of the Manual Trigger node to the input of the HTTP Request node.

3. **Install and Configure CustomJS PDF Toolkit Node:**  
   - Ensure your n8n instance is self-hosted and install the community node package `@custom-js/n8n-nodes-pdf-toolkit` if not already installed.  
   - Add the “Extract Pages” node from this package.  
   - Set the `pageRange` parameter to the pages you want to extract, e.g., `2-3`.  
   - Set the `field_name` parameter to `=data` (this references the PDF binary data field from the HTTP Request).  
   - Create and assign credentials for the CustomJS API key:  
     - Go to n8n credentials and create a new credential for CustomJS API.  
     - Paste your API key obtained from https://www.customjs.space.  
   - Assign these credentials to the Extract Pages node.  
   - Connect the output of the HTTP Request node to the input of the Extract Pages node.

4. **Save and Activate Workflow:**  
   - Verify connections: Manual Trigger → HTTP Request → Extract Pages.  
   - Optionally, add nodes to output or save the extracted PDF pages (e.g., Write Binary File, Webhook Response).  
   - Execute the workflow manually or trigger as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Community nodes like CustomJS PDF Toolkit can only be installed on self-hosted n8n instances.                                  | Official documentation and npm package: [@custom-js/n8n-nodes-pdf-toolkit](https://www.npmjs.com/package/@custom-js/n8n-nodes-pdf-toolkit) |
| Obtain API key by signing up at customJS and retrieving it from your profile page.                                             | https://www.customjs.space                                                                        |
| This workflow can be adapted by replacing the manual trigger with a webhook or other triggers for automation or API integration. |                                                                                                 |
| Recommended to handle possible HTTP and API errors when downloading or processing PDFs to ensure workflow robustness.          |                                                                                                 |
| Perfect for extracting specific pages or segments from PDF documents for reporting, notes, or splitting workflows.             |                                                                                                 |

---

This documentation thoroughly covers the entire workflow, enabling full understanding, modification, and reproduction without access to the original JSON.