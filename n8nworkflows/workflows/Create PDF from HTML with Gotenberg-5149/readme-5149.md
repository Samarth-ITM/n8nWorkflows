Create PDF from HTML with Gotenberg

https://n8nworkflows.xyz/workflows/create-pdf-from-html-with-gotenberg-5149


# Create PDF from HTML with Gotenberg

### 1. Workflow Overview

This workflow automates the conversion of HTML content into a PDF file using the Gotenberg API. It is designed for users needing to programmatically generate PDF documents from HTML input, specifying the output file name dynamically. The workflow is logically divided into three blocks:

- **1.1 Input Reception:** Receives JSON input containing HTML code and desired PDF file name.
- **1.2 HTML File Creation:** Converts the HTML string into a file format suitable for upload.
- **1.3 PDF Conversion via Gotenberg:** Sends the HTML file to a Gotenberg service instance for conversion to PDF, applying metadata and naming conventions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This initial block receives and triggers the workflow with the necessary input data: raw HTML content and the target PDF file name (without extension). It provides a JSON example for easy testing and clarity on expected input structure.

- **Nodes Involved:**  
  - Create PDF from HTML

- **Node Details:**  
  - **Create PDF from HTML**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point to receive JSON input with fields `html` and `file_name`.  
    - Configuration: Uses `jsonExample` input source with sample JSON:  
      ```json
      {
        "html": "<h1>Hello World</h1>",
        "file_name": "my-report-2024"
      }
      ```  
    - Input: JSON object specifying HTML content and filename base  
    - Output: Passes JSON data downstream  
    - Edge Cases:  
      - Missing or malformed JSON input could cause errors or empty file generation.  
      - Absence of either `html` or `file_name` fields may break subsequent nodes.  
    - Notes: Sticky Note adjacent explains input expectations and sample.

#### 1.2 HTML File Creation

- **Overview:**  
  Converts the raw HTML string from input into a file object, making it ready for multipart/form-data upload to the PDF conversion API.

- **Nodes Involved:**  
  - Create index.html

- **Node Details:**  
  - **Create index.html**  
    - Type: Convert To File  
    - Role: Converts the `html` property from input JSON into a UTF-8 encoded text file named `index.html`.  
    - Configuration:  
      - Operation: `toText`  
      - Source Property: `html` (from input JSON)  
      - Encoding: UTF-8  
      - Filename: `index.html` (static)  
    - Input: Receives JSON with `html` field  
    - Output: Binary file named `index.html` containing the HTML content  
    - Edge Cases:  
      - If `html` is empty or invalid, output file will be empty or malformed.  
      - Encoding issues if input contains non-UTF-8 characters (rare).  
    - Connection: Passes binary file to HTTP Request node for upload.

#### 1.3 PDF Conversion via Gotenberg

- **Overview:**  
  Sends the created HTML file as a multipart form upload to a locally hosted or network-accessible Gotenberg service endpoint to convert it into a PDF file. Metadata fields are set dynamically, and the output PDF is named as per input.

- **Nodes Involved:**  
  - Convert to PDF with Gotenberg  
  - Sticky Note (Gotenberg API Call details)

- **Node Details:**  
  - **Convert to PDF with Gotenberg**  
    - Type: HTTP Request  
    - Role: HTTP POST request to Gotenberg API `/forms/chromium/convert/html` to convert HTML into PDF.  
    - Configuration:  
      - URL: `http://gotenberg:3000/forms/chromium/convert/html` (default assumes Docker network with Gotenberg)  
      - Method: POST  
      - Content-Type: `multipart-form-data`  
      - Body Parameters:  
        - `form file`: binary file from previous node (`Create index.html`)  
        - `scale`: `"1"` (default PDF scale)  
        - `metadata`: JSON string dynamically generated with current date and static fields (Author, Title, Producer, etc.) using expression:  
          ```json
          {
            "Author": "IA2S",
            "Copyright": "IA2S",
            "CreateDate": "{{ $now.format('yyyy-MM-dd') }}",
            "Creator": "IA2S",
            "Keywords": [],
            "ModDate": "{{ $now.format('yyyy-MM-dd') }}",
            "PDFVersion": 1.7,
            "Producer": "IA2S",
            "Subject": "PDF",
            "Title": "IA2S PDF"
          }
          ```  
      - Headers:  
        - `Gotenberg-Output-Filename`: dynamically set to the input `file_name` property (e.g., `my-report-2024`)  
    - Input: Multipart form data including binary HTML file  
    - Output: PDF binary file stream returned from Gotenberg  
    - Edge Cases:  
      - Connection errors if Gotenberg service is unreachable or URL is incorrect.  
      - Timeout or HTTP error responses if Gotenberg fails to convert.  
      - Metadata JSON malformed expressions could cause API to reject request.  
      - Filename header missing or malformed may cause incorrect output file naming.  
    - Notes: Sticky note explains URL dependency and metadata customization.

- **Sticky Note (Gotenberg API Call):**  
  - Explains default URL assumption and instructions to change if Gotenberg runs outside Docker.  
  - Highlights ability to customize PDF metadata.

---

### 3. Summary Table

| Node Name               | Node Type                | Functional Role                      | Input Node(s)        | Output Node(s)            | Sticky Note                                                                                                                             |
|-------------------------|--------------------------|------------------------------------|----------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Create PDF from HTML     | Execute Workflow Trigger | Receives input JSON with HTML & filename | None                 | Create index.html          | Explains required input JSON format and provides example.                                                                               |
| Create index.html        | Convert To File          | Converts HTML string to UTF-8 file | Create PDF from HTML  | Convert to PDF with Gotenberg |                                                                                                                                          |
| Convert to PDF with Gotenberg | HTTP Request            | Sends HTML file to Gotenberg API for PDF conversion | Create index.html     | None                      | Explains Gotenberg URL default, requirement to adjust if hosted externally, and metadata customization options.                         |
| Sticky Note             | Sticky Note               | Documentation and explanation       | None                 | None                      | Covers input JSON format, Gotenberg service prerequisite, and API call details.                                                          |
| Sticky Note1            | Sticky Note               | Documentation on input and configuration | None                 | None                      | Details the expected input JSON fields.                                                                                                |
| Sticky Note2            | Sticky Note               | Documentation on prerequisite Gotenberg service | None                 | None                      | Explains requirement to have a running Gotenberg service with example Docker Compose snippet.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add an **Execute Workflow Trigger** node named `Create PDF from HTML`.  
   - Set `Input Source` to `jsonExample`.  
   - Paste example JSON input:  
     ```json
     {
       "html": "<h1>Hello World</h1>",
       "file_name": "my-report-2024"
     }
     ```  
   - This node starts the workflow and provides sample data for testing.

2. **Add Convert To File Node**  
   - Add a **Convert To File** node named `Create index.html`.  
   - Set operation to `toText`.  
   - Set `Source Property` to `html` (this reads the HTML string from input JSON).  
   - Under options, set encoding to `utf8`.  
   - Set filename to `index.html` (static).  
   - Connect `Create PDF from HTML` output to `Create index.html` input.

3. **Add HTTP Request Node for Gotenberg**  
   - Add an **HTTP Request** node named `Convert to PDF with Gotenberg`.  
   - Set method to `POST`.  
   - Set URL to `http://gotenberg:3000/forms/chromium/convert/html`.  
     - **Note:** Change this URL if your Gotenberg instance is not accessible at this address.  
   - Set content type to `multipart-form-data`.  
   - Enable sending body and headers.  
   - Under Body Parameters, add:  
     - Parameter name: `form file`  
       - Type: `formBinaryData`  
       - Input Data Field Name: `data` (binds to binary file from `Create index.html`)  
     - Parameter name: `scale` with value `"1"`  
     - Parameter name: `metadata` with value set as expression:  
       ```
       ={"Author":"IA2S","Copyright":"IA2S","CreateDate":"{{$now.format('yyyy-MM-dd')}}","Creator":"IA2S","Keywords":[],"ModDate":"{{$now.format('yyyy-MM-dd')}}","PDFVersion":1.7,"Producer":"IA2S","Subject":"PDF","Title":"IA2S PDF"}
       ```  
   - Under Headers, add:  
     - Header name: `Gotenberg-Output-Filename`  
     - Value: expression referencing input filename property: `={{ $('Create PDF from HTML').last().json.file_name }}`  
   - Connect `Create index.html` output to this HTTP Request node.

4. **Optional: Add Sticky Notes**  
   - Add sticky notes to document:  
     - Input expectations and sample JSON for the trigger node.  
     - Instructions about running Gotenberg service (e.g., Docker Compose snippet).  
     - Details on API call customization and URL adjustment.

5. **Credentials**  
   - No special credentials needed for this workflow unless your Gotenberg instance requires authentication.  
   - If authentication is required, configure HTTP Request node accordingly.

6. **Testing**  
   - Run the workflow manually with the example input to verify that a PDF file named as per `file_name` is generated and returned.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow requires a running Gotenberg instance to convert HTML to PDF.                                    | Gotenberg Docker image: `gotenberg/gotenberg:8` and official docs at https://gotenberg.dev          |
| Default Gotenberg URL `http://gotenberg:3000` assumes n8n and Gotenberg run on the same Docker network.         | Adjust URL in HTTP Request node if Gotenberg is hosted externally.                                   |
| PDF metadata fields (Author, Title, etc.) can be customized in the HTTP Request node's body parameters.        | Useful for branding, document info, or compliance metadata.                                         |
| Example `docker-compose.yml` snippet provided to run Gotenberg as a service.                                   | Included in Sticky Note2 for quick setup.                                                           |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.