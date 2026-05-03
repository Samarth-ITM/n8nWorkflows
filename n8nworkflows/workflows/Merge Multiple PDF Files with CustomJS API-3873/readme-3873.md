Merge Multiple PDF Files with CustomJS API

https://n8nworkflows.xyz/workflows/merge-multiple-pdf-files-with-customjs-api-3873


# Merge Multiple PDF Files with CustomJS API

### 1. Workflow Overview

This workflow automates the process of downloading multiple PDF files from public URLs, merging them into a single PDF document, and saving the result to disk. It is designed for self-hosted n8n instances using a CustomJS API key to leverage the PDF Toolkit node for merging PDFs. The workflow is ideal for bundling reports, invoices, or any sets of PDF documents from external sources without custom coding.

The logical structure of the workflow is organized into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 PDF Download:** Two HTTP Request nodes to download PDF files from given URLs.
- **1.3 Array Population:** Merge node to combine multiple downloaded PDF binary files into an array.
- **1.4 PDF Merging:** CustomJS Merge PDF node to merge the array of PDF files into one.
- **1.5 File Output:** Write merged PDF to disk, then read it back for further use or verification.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, allowing user control over when the PDF merging process begins.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow upon user interaction in the n8n editor or UI.  
    - Configuration: Default manual trigger, no parameters needed.  
    - Inputs: None  
    - Outputs: Triggers the subsequent HTTP Request nodes for PDF downloads.  
    - Edge cases: None specific; workflow will not start unless triggered manually.

#### 1.2 PDF Download

- **Overview:**  
  Downloads two PDF files from publicly accessible URLs using HTTP requests.

- **Nodes Involved:**  
  - HTTP Request  
  - HTTP Request2

- **Node Details:**  

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Downloads the first PDF file from a specified URL.  
    - Configuration:  
      - URL: `https://www.intewa.com/fileadmin/documents/pdf-file.pdf` (expression used for flexibility)  
      - Method: GET (default)  
      - Response Format: Binary (implied by use in merging)  
      - Options: Default, no authentication  
    - Inputs: Triggered by Manual Trigger  
    - Outputs: Binary data of the first PDF file  
    - Edge cases:  
      - URL unreachable or file not found (404)  
      - Network timeout or connection errors  
      - Non-PDF content returned

  - **HTTP Request2**  
    - Type: HTTP Request  
    - Role: Downloads the second PDF file from another public URL.  
    - Configuration:  
      - URL: `https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf` (expression)  
      - Method: GET  
      - Response Format: Binary  
      - Options: Default, no authentication  
    - Inputs: Triggered by Manual Trigger (parallel to HTTP Request)  
    - Outputs: Binary data of the second PDF file  
    - Edge cases: Same as HTTP Request node

#### 1.3 Array Population

- **Overview:**  
  Combines the two downloaded PDF files into a single array to be passed to the merge operation.

- **Nodes Involved:**  
  - Merge

- **Node Details:**  

  - **Merge**  
    - Type: Merge node  
    - Role: Combines inputs from two HTTP Request nodes into an array with each PDF as an element.  
    - Configuration: Default merge mode (likely 'Wait' or 'Merge by position') to combine inputs from both HTTP nodes.  
    - Inputs: Receives two input streams from HTTP Request and HTTP Request2 nodes (index 0 and 1)  
    - Outputs: Single output containing an array of two PDF binary files  
    - Edge cases:  
      - One input missing or delayed could cause workflow to wait indefinitely (depends on merge mode)  
      - Data format inconsistency if HTTP Request nodes do not return binary PDF data  

#### 1.4 PDF Merging

- **Overview:**  
  Uses the CustomJS PDF Toolkit to merge multiple PDF files supplied as an array into a single PDF document.

- **Nodes Involved:**  
  - Merge PDF1

- **Node Details:**  

  - **Merge PDF1**  
    - Type: @custom-js/n8n-nodes-pdf-toolkit.mergePdfs  
    - Role: Merges multiple PDF binary files into one PDF file.  
    - Configuration: Default; uses credentials for CustomJS API integration.  
    - Credentials: CustomJS API key (configured via “CustomJS account” credential)  
    - Inputs: Receives array of PDFs from Merge node  
    - Outputs: Binary data of merged PDF  
    - Edge cases:  
      - API key invalid or expired causing authentication failure  
      - Exceeding file size or number limits imposed by CustomJS API  
      - Network errors or API downtime  
      - Input data not properly formatted as PDFs  
    - Note: If total PDF size > 6MB, alternative approach suggested (pass array of URLs instead of binaries).

#### 1.5 File Output

- **Overview:**  
  Saves the merged PDF file to disk and then reads it back, allowing persistence and further processing.

- **Nodes Involved:**  
  - Read/Write Files from Disk (write operation)  
  - Read/Write Files from Disk4 (read operation)

- **Node Details:**  

  - **Read/Write Files from Disk**  
    - Type: Read/Write File  
    - Role: Writes the merged PDF binary data to a file named `test.pdf` on disk.  
    - Configuration:  
      - Operation: Write  
      - File Name: `test.pdf` (fixed name)  
      - Data Source: Binary data from Merge PDF1 node  
    - Inputs: Merged PDF binary data  
    - Outputs: Confirmation or metadata of written file  
    - Edge cases:  
      - File system permission errors  
      - Insufficient disk space  
      - Filename conflicts or invalid paths  

  - **Read/Write Files from Disk4**  
    - Type: Read/Write File  
    - Role: Reads the previously saved `test.pdf` from disk.  
    - Configuration:  
      - Operation: Read (implied by fileSelector)  
      - File Selector: `test.pdf`  
    - Inputs: Triggered by output of write node  
    - Outputs: Binary data of the saved merged PDF  
    - Edge cases:  
      - File not found or deleted before read  
      - Permission errors  
      - Disk errors

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                         | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                                                                                                |
|----------------------------|-------------------------------------|---------------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                      | Start workflow on user action         | None                         | HTTP Request, HTTP Request2   |                                                                                                                                                                                                            |
| HTTP Request               | HTTP Request                        | Download first PDF file                | When clicking ‘Test workflow’ | Merge                         |                                                                                                                                                                                                            |
| HTTP Request2              | HTTP Request                        | Download second PDF file               | When clicking ‘Test workflow’ | Merge                         |                                                                                                                                                                                                            |
| Merge                     | Merge                              | Combine multiple PDFs into array      | HTTP Request, HTTP Request2  | Merge PDF1                    |                                                                                                                                                                                                            |
| Merge PDF1                | CustomJS PDF Toolkit (mergePdfs)   | Merge PDF files into one PDF           | Merge                        | Read/Write Files from Disk    | Requires CustomJS API key credential. Community nodes only supported on self-hosted n8n. If PDF size >6MB, passing URLs instead of binaries is recommended. See https://www.npmjs.com/package/@custom-js/n8n-nodes-pdf-toolkit |
| Read/Write Files from Disk | Read/Write File                    | Write merged PDF to disk               | Merge PDF1                   | Read/Write Files from Disk4   |                                                                                                                                                                                                            |
| Read/Write Files from Disk4| Read/Write File                    | Read merged PDF from disk              | Read/Write Files from Disk   | None                         |                                                                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No configuration needed.

2. **Create First HTTP Request Node:**  
   - Add an **HTTP Request** node named `HTTP Request`.  
   - Set **URL** parameter to `https://www.intewa.com/fileadmin/documents/pdf-file.pdf` using an expression (e.g., `"=https://www.intewa.com/fileadmin/documents/pdf-file.pdf"`).  
   - Use default method GET.  
   - Ensure response is set to binary format for file download.

3. **Create Second HTTP Request Node:**  
   - Add a second **HTTP Request** node named `HTTP Request2`.  
   - Set **URL** to `https://www.w3.org/WAI/ER/tests/xhtml/testfiles/resources/pdf/dummy.pdf` (expression).  
   - Default GET method, binary response.

4. **Connect Manual Trigger to Both HTTP Requests:**  
   - From `When clicking ‘Test workflow’` node, connect output to both `HTTP Request` and `HTTP Request2` nodes (parallel connections).

5. **Add Merge Node:**  
   - Add a **Merge** node named `Merge`.  
   - Connect outputs of `HTTP Request` and `HTTP Request2` to inputs 0 and 1 of `Merge` node respectively.  
   - Use default merge mode (typically "Merge By Position" or "Wait").

6. **Add CustomJS Merge PDFs Node:**  
   - Install and configure the **@custom-js/n8n-nodes-pdf-toolkit** package on your self-hosted n8n instance.  
   - Add a **Merge PDF1** node of type `mergePdfs` from the CustomJS package.  
   - Configure credentials: Create and assign a CustomJS API key credential (see [customJS](https://www.customjs.space) to get API key).  
   - Connect `Merge` node output to `Merge PDF1` input.

7. **Add Write to Disk Node:**  
   - Add a **Read/Write Files from Disk** node named `Read/Write Files from Disk`.  
   - Configure operation to `write`.  
   - Set file name to `test.pdf`.  
   - Connect output of `Merge PDF1` to this node.

8. **Add Read from Disk Node:**  
   - Add another **Read/Write Files from Disk** node named `Read/Write Files from Disk4`.  
   - Configure operation to `read` with file selector set to `test.pdf`.  
   - Connect output of the write node to this read node.

9. **Workflow Finalization:**  
   - Verify all connections and node parameters.  
   - Activate the workflow and test by triggering manually.  
   - Observe merged PDF saved as `test.pdf` on disk.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Community nodes such as the CustomJS PDF Toolkit can only be installed on self-hosted n8n instances.                   | https://www.npmjs.com/package/@custom-js/n8n-nodes-pdf-toolkit                                           |
| To obtain the CustomJS API key, sign up on the customJS platform and retrieve the key from your profile page.         | https://www.customjs.space                                                                              |
| For workflows with large PDFs (>6MB), it is recommended to pass arrays of URLs to the merge node rather than binaries. | PDF Toolkit usage note                                                                                   |
| The workflow can be adapted to trigger via webhook and provide the merged PDF as a webhook response instead of disk.   | Suggested alternative use case                                                                            |
| Screenshots in the original workflow description illustrate API key retrieval and setting credentials in n8n.          | Attached images in the original documentation                                                           |

---

This document provides a complete and clear reference for understanding, reproducing, and maintaining the "Merge Multiple PDF Files with CustomJS API" n8n workflow.