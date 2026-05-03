Convert PPTX to PDF using ConvertAPI

https://n8nworkflows.xyz/workflows/convert-pptx-to-pdf-using-convertapi-2305


# Convert PPTX to PDF using ConvertAPI

### 1. Workflow Overview

This workflow automates the conversion of a PPTX presentation file to a PDF document using the ConvertAPI service. It is designed for developers and organizations needing reliable file format transformation within automated processes. The workflow encapsulates three primary logical blocks:

- **1.1 Input Trigger and File Download:** Initiates the workflow manually and downloads the source PPTX file from a public URL.
- **1.2 File Conversion:** Sends the downloaded PPTX file to ConvertAPI for conversion to PDF, handling authentication securely.
- **1.3 Output Storage:** Saves the converted PDF file to the local file system.

This modular design allows easy customization for different input sources, output destinations, or file types by adjusting node parameters.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and File Download

**Overview:**  
This block starts the workflow manually and fetches the PPTX file from a remote URL to be processed.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Download PPTX File  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - **Type:** Manual Trigger  
  - **Role:** Initiates the workflow on demand, allowing testing or manual runs.  
  - **Configuration:** Default manual trigger with no parameters.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Triggers "Download PPTX File" node.  
  - **Edge Cases:** None typical; workflow will not start without manual trigger.  

- **Download PPTX File**  
  - **Type:** HTTP Request  
  - **Role:** Downloads the PPTX file from a specified URL.  
  - **Configuration:**  
    - URL set to `https://cdn.convertapi.com/public/files/demo.pptx` (public demo file).  
    - Response format set to file download.  
    - HTTP method: GET (default).  
  - **Expressions:** None used; URL is static but can be parameterized.  
  - **Inputs:** Triggered by manual trigger node.  
  - **Outputs:** Passes binary file data to "File conversion to PDF".  
  - **Edge Cases:**  
    - Network failures or URL unreachable.  
    - File not found or changed at source URL.  
    - Large file downloads timing out.  

#### 1.2 File Conversion

**Overview:**  
This block sends the downloaded PPTX file to ConvertAPI’s PPTX-to-PDF conversion endpoint, authenticating via query parameters, and receives the converted PDF as a file response.

**Nodes Involved:**  
- File conversion to PDF  
- Sticky Note (documentation)  

**Node Details:**

- **File conversion to PDF**  
  - **Type:** HTTP Request  
  - **Role:** Converts the PPTX file to PDF via ConvertAPI.  
  - **Configuration:**  
    - URL: `https://v2.convertapi.com/convert/pptx/to/pdf`.  
    - Method: POST.  
    - Content type: multipart/form-data to send binary file data.  
    - Authentication: Query parameter authentication with ConvertAPI secret stored in credentials (Query Auth account).  
    - Header: Accepts application/octet-stream for binary PDF response.  
    - Body Parameters: Includes the binary data field named "file" from previous node output.  
  - **Expressions:** Uses `=data` to reference binary data from prior node.  
  - **Inputs:** Receives binary file from "Download PPTX File".  
  - **Outputs:** Passes converted PDF binary file to "Write Result File to Disk".  
  - **Edge Cases:**  
    - Invalid or missing authentication secret leading to 401 errors.  
    - API downtime or rate limiting by ConvertAPI.  
    - Incorrect file data causing conversion errors.  
    - Response format mismatches or corrupted files.  
  - **Notes:** Authentication information linked in the sticky note for user reference.  

- **Sticky Note**  
  - **Type:** Sticky Note (documentation only)  
  - **Content:** Explains authentication requirement and provides link to ConvertAPI account creation for secret key.  
  - **Role:** User guidance, no functional role in workflow execution.  

#### 1.3 Output Storage

**Overview:**  
This block writes the converted PDF binary data to a file named `document.pdf` on the local disk.

**Nodes Involved:**  
- Write Result File to Disk  

**Node Details:**

- **Write Result File to Disk**  
  - **Type:** Read/Write File  
  - **Role:** Writes binary PDF data to a local file.  
  - **Configuration:**  
    - Operation: Write.  
    - File name: `document.pdf`.  
    - Data source: Uses expression `=data` to write the binary data from previous node.  
  - **Inputs:** Receives converted PDF binary from "File conversion to PDF".  
  - **Outputs:** None (final node).  
  - **Edge Cases:**  
    - File system permissions preventing write.  
    - Disk space limitations.  
    - Invalid or corrupted binary data causing file write errors.  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                      |
|---------------------------|---------------------|------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Initiates workflow manually        | None                        | Download PPTX File          |                                                                                                |
| Download PPTX File        | HTTP Request        | Downloads PPTX file from URL       | When clicking ‘Test workflow’ | File conversion to PDF      |                                                                                                |
| File conversion to PDF    | HTTP Request        | Converts PPTX to PDF via API       | Download PPTX File           | Write Result File to Disk   | ## Authentication<br>Conversion requests must be authenticated. Please create  [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin) |
| Write Result File to Disk | Read/Write File     | Saves converted PDF to disk        | File conversion to PDF       | None                       |                                                                                                |
| Sticky Note              | Sticky Note         | Authentication info and guidance   | None                        | None                       | ## Authentication<br>Conversion requests must be authenticated. Please create  [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Start workflow manually.

2. **Create an HTTP Request node to download PPTX**  
   - Name: `Download PPTX File`  
   - Connect input from `When clicking ‘Test workflow’`.  
   - Set HTTP Method: GET (default).  
   - Set URL: `https://cdn.convertapi.com/public/files/demo.pptx` (or your source URL).  
   - Under Options, set Response Format to `File` to download binary data.

3. **Create an HTTP Request node for file conversion**  
   - Name: `File conversion to PDF`  
   - Connect input from `Download PPTX File`.  
   - Set HTTP Method: POST.  
   - Set URL: `https://v2.convertapi.com/convert/pptx/to/pdf`.  
   - Under Parameters:  
     - Set Content Type: `multipart/form-data`.  
     - Enable sending Headers.  
     - Add Header `Accept` with value `application/octet-stream`.  
   - Under Body Parameters:  
     - Add parameter with:  
       - Name: `file`  
       - Type: `Form Binary Data`  
       - Input Data Field Name: `=data` (to pass binary from previous node).  
   - Configure Authentication:  
     - Add HTTP Query Authentication credentials with your ConvertAPI secret key as query parameter (e.g., `Secret=<your_secret>`).  
     - Use or create credentials named `Query Auth account` in n8n.  
   - Set Response Format to `File` to receive binary PDF data.

4. **Add a Read/Write File node to save PDF**  
   - Name: `Write Result File to Disk`  
   - Connect input from `File conversion to PDF`.  
   - Operation: Write.  
   - File Name: `document.pdf`.  
   - Data Property Name: `=data` (binary PDF data from previous node).  
   - Confirm local file system permissions are granted.

5. **(Optional) Add a Sticky Note for authentication instructions**  
   - Content:  
     ```
     ## Authentication
     Conversion requests must be authenticated. Please create
     [ConvertAPI account to get authentication secret](https://www.convertapi.com/a/signin)
     ```
   - Place near the "File conversion to PDF" node for reference.

6. **Connect nodes in sequence:**  
   `When clicking ‘Test workflow’` → `Download PPTX File` → `File conversion to PDF` → `Write Result File to Disk`

7. **Test the workflow by running the manual trigger**.  
   - Verify that `document.pdf` appears locally as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Conversion requests must be authenticated via a secret key obtained from ConvertAPI.                                       | https://www.convertapi.com/a/signin                          |
| All ConvertAPI endpoints and documentation are available at https://www.convertapi.com/api                                 | https://www.convertapi.com/api                               |
| The workflow demonstrates usage of multipart/form-data HTTP requests to upload files for conversion.                       | n8n HTTP Request Node Documentation                          |
| Adjust the URL parameter in the Download PPTX File node to convert files from other sources or URLs.                        | Customization instruction in workflow description            |

---

This documentation provides a comprehensive understanding of the workflow’s design, configuration, and usage, enabling effective reproduction, troubleshooting, and customization.