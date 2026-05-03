Convert PDF to HTML using PDF.co and Google Drive

https://n8nworkflows.xyz/workflows/convert-pdf-to-html-using-pdf-co-and-google-drive-3275


# Convert PDF to HTML using PDF.co and Google Drive

### 1. Workflow Overview

This workflow automates the conversion of newly uploaded PDF files in a specified Google Drive folder into HTML format using the PDF.co API. Upon detecting a new PDF file, it validates the file type, sends the PDF to PDF.co for conversion, processes the API response into a binary HTML file, and uploads the resulting HTML back to Google Drive. This automation eliminates manual intervention in document conversion, streamlining workflows for web publishing, archiving, or further processing.

**Logical Blocks:**

- **1.1 Input Reception:** Detect new PDF files in a specific Google Drive folder.
- **1.2 File Validation:** Confirm that the detected file is a PDF.
- **1.3 PDF to HTML Conversion:** Send the PDF to PDF.co API and receive HTML output.
- **1.4 Response Processing:** Convert the API response into a binary file suitable for upload.
- **1.5 Output Storage:** Upload the converted HTML file back to Google Drive.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block monitors a designated Google Drive folder and triggers the workflow whenever a new file is created.

- **Nodes Involved:**  
  - Google Drive Trigger

- **Node Details:**

  - **Google Drive Trigger**  
    - **Type:** Trigger node for Google Drive events  
    - **Configuration:**  
      - Event: `fileCreated` (trigger on new files)  
      - Polling interval: every minute  
      - Folder to watch: a specific folder selected by URL (must be configured)  
    - **Key Expressions:** None  
    - **Input:** None (trigger node)  
    - **Output:** Emits metadata of the newly created file, including MIME type and webViewLink  
    - **Version Requirements:** n8n version supporting Google Drive Trigger node (version 1 used here)  
    - **Potential Failures:**  
      - Authentication errors if Google Drive OAuth2 credentials are invalid or expired  
      - Folder access permission issues  
      - Polling delays or missed events if API limits are reached  

#### 1.2 File Validation

- **Overview:**  
  This block filters incoming files to ensure only PDFs proceed to conversion.

- **Nodes Involved:**  
  - If Node

- **Node Details:**

  - **If Node**  
    - **Type:** Conditional node  
    - **Configuration:**  
      - Condition: Check if `mimeType` equals `application/pdf` (case-sensitive, strict type validation)  
    - **Key Expressions:**  
      - `={{ $json.mimeType }}` compared to `"application/pdf"`  
    - **Input:** Output from Google Drive Trigger  
    - **Output:**  
      - `true` branch: file is a PDF, proceed  
      - `false` branch: file is not a PDF, workflow stops here  
    - **Version Requirements:** Version 2.2 used for advanced condition options  
    - **Potential Failures:**  
      - Missing or malformed `mimeType` field in input JSON  
      - Case sensitivity causing false negatives if MIME type varies  
      - Workflow halts silently on false condition (no error, but no further processing)  

#### 1.3 PDF to HTML Conversion

- **Overview:**  
  Sends the PDF file URL to PDF.co API to convert the PDF into HTML format.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - **Type:** HTTP POST request node  
    - **Configuration:**  
      - URL: `https://api.pdf.co/v1/pdf/convert/to/html`  
      - Method: POST  
      - Authentication: HTTP Header Auth with PDF.co API key  
      - Headers: Contains API key header (configured via credentials)  
      - Body Parameters:  
        - `url`: uses `{{$json.webViewLink}}` from Google Drive file metadata (public or accessible URL of the PDF)  
        - `inline`: `"true"` (request inline HTML output)  
        - `async`: `false` (synchronous processing)  
        - Other parameters like `pages`, `unwrap`, `name`, `password`, `lineGrouping`, `profiles` are included but mostly empty or default  
      - Sends body and headers as configured  
    - **Key Expressions:**  
      - `{{$json.webViewLink}}` to pass the PDF URL to the API  
    - **Input:** Output from If Node (only PDFs)  
    - **Output:** API response containing HTML content in the body  
    - **Version Requirements:** Version 4.2 for HTTP Request node  
    - **Potential Failures:**  
      - Authentication failure if API key is invalid or missing  
      - Network timeouts or API rate limits  
      - Invalid or inaccessible PDF URL causing API errors  
      - API response format changes or errors not handled explicitly  
    - **Credential:** HTTP Header Auth with PDF.co API key  

#### 1.4 Response Processing

- **Overview:**  
  Converts the HTML string returned by the API into a binary format suitable for file upload.

- **Nodes Involved:**  
  - Function (Code) Node named "Convert to Binary File"

- **Node Details:**

  - **Convert to Binary File**  
    - **Type:** Code node (JavaScript)  
    - **Configuration:**  
      - JavaScript code converts the API response body (HTML string) into a Buffer, then encodes it as base64 binary data with MIME type `text/html` and filename `sample.html`  
    - **Key Code Snippet:**  
      ```js
      const buffer = Buffer.from($json.body, 'utf-8');
      return [
        {
          binary: {
            data: {
              data: buffer.toString('base64'),
              mimeType: 'text/html',
              fileName: 'sample.html'
            }
          }
        }
      ];
      ```  
    - **Input:** API response JSON from HTTP Request node  
    - **Output:** Binary data object with HTML file content  
    - **Version Requirements:** Version 2 for Code node  
    - **Potential Failures:**  
      - Missing or malformed `body` property in API response  
      - Encoding errors if response is not valid UTF-8 string  
      - Filename hardcoded as `sample.html` (may cause overwrite if multiple files processed simultaneously)  

#### 1.5 Output Storage

- **Overview:**  
  Uploads the converted HTML binary file back to a specified Google Drive folder.

- **Nodes Involved:**  
  - Google Drive (Upload File)

- **Node Details:**

  - **Google Drive**  
    - **Type:** Google Drive node with Upload File action  
    - **Configuration:**  
      - Action: Upload File  
      - Destination folder: specified by folder ID or URL (must be configured)  
      - File name: `sample.html` (mapped from binary data)  
      - Binary property: mapped from previous Function node output (`binary.data`)  
    - **Key Expressions:** None explicitly, but folder ID and drive ID are URL-based and require proper configuration  
    - **Input:** Binary data from "Convert to Binary File" node  
    - **Output:** Metadata of uploaded file  
    - **Version Requirements:** Version 3 for Google Drive node  
    - **Potential Failures:**  
      - Authentication errors with Google Drive OAuth2  
      - Insufficient permissions to upload to target folder  
      - File overwrite if multiple files named `sample.html` are uploaded to the same folder  
      - Folder ID or Drive ID misconfiguration causing upload failure  

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                  | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                          |
|-----------------------|----------------------------|---------------------------------|------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Google Drive Trigger   | Google Drive Trigger       | Detect new files in folder      | —                      | If                       |                                                                                                    |
| If                    | If                         | Check if file is PDF            | Google Drive Trigger   | HTTP Request             |                                                                                                    |
| HTTP Request          | HTTP Request               | Convert PDF to HTML via API     | If                     | Convert to Binary File   |                                                                                                    |
| Convert to Binary File | Code (Function)            | Convert API response to binary  | HTTP Request            | Google Drive             |                                                                                                    |
| Google Drive          | Google Drive (Upload File) | Upload converted HTML to Drive  | Convert to Binary File  | —                        |                                                                                                    |
| Sticky Note           | Sticky Note                | Workflow title                  | —                      | —                        | ## Automated PDF to HTML Conversion                                                                |
| Sticky Note1          | Sticky Note                | Workflow description            | —                      | —                        | ## Description: This n8n workflow automates the process of converting a newly stored PDF file...   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Add node: Google Drive Trigger  
   - Set event to `fileCreated`  
   - Authenticate with Google OAuth2 credentials  
   - Select the specific folder to monitor (by URL or folder picker)  
   - Set polling interval to every minute  
   - Save and test trigger  

2. **Add If Node for PDF Validation**  
   - Add node: If  
   - Configure condition:  
     - Left value: `{{$json.mimeType}}`  
     - Operation: equals  
     - Right value: `application/pdf`  
     - Case sensitive: true  
     - Type validation: strict  
   - Connect Google Drive Trigger output to If node input  
   - Save and test  

3. **Add HTTP Request Node for PDF.co Conversion**  
   - Add node: HTTP Request  
   - Set method to POST  
   - Set URL to `https://api.pdf.co/v1/pdf/convert/to/html`  
   - Authentication: HTTP Header Auth with PDF.co API key (create credential with your API key)  
   - Headers: include API key header as required by PDF.co  
   - Body parameters (form or JSON):  
     - `url`: `{{$json.webViewLink}}` (URL of the PDF file)  
     - `inline`: `true`  
     - `async`: `false`  
     - Other parameters can be left default or empty  
   - Connect If node’s true output to HTTP Request node input  
   - Save and test  

4. **Add Function Node to Convert Response to Binary**  
   - Add node: Function (Code)  
   - Paste the following JavaScript code:  
     ```js
     const buffer = Buffer.from($json.body, 'utf-8');
     return [
       {
         binary: {
           data: {
             data: buffer.toString('base64'),
             mimeType: 'text/html',
             fileName: 'sample.html'
           }
         }
       }
     ];
     ```  
   - Connect HTTP Request node output to this node  
   - Save and test  

5. **Add Google Drive Node to Upload HTML File**  
   - Add node: Google Drive  
   - Set action to Upload File  
   - Authenticate with Google OAuth2 credentials  
   - Set destination folder by folder ID or URL  
   - Map binary property: select `data` from previous Function node output  
   - Set file name as `sample.html` (or parameterize if needed)  
   - Connect Function node output to Google Drive node input  
   - Save and test  

6. **Connect Nodes in Order**  
   - Google Drive Trigger → If → HTTP Request → Convert to Binary File → Google Drive Upload  

7. **Test the Workflow**  
   - Activate the workflow  
   - Upload a PDF file to the monitored Google Drive folder  
   - Verify that the workflow triggers, converts the PDF to HTML, and uploads the HTML file back to Google Drive  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow automates PDF to HTML conversion triggered by new Google Drive files.                                | Workflow purpose                                                                                       |
| Requires PDF.co API key and Google Drive OAuth2 authentication.                                               | Prerequisites                                                                                        |
| Customization options include converting to other formats, filtering files by size, and sending notifications.| Customization suggestions                                                                              |
| Developed by WeblineIndia’s AI development team.                                                              | https://www.weblineindia.com/ai-development.html                                                      |
| Over 3500 software projects delivered across 25+ countries since 1999.                                        | Company background                                                                                     |
| Hire AI developers via WeblineIndia.                                                                           | https://www.weblineindia.com/hire-ai-developers.html                                                  |

---

This structured documentation provides a complete understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.