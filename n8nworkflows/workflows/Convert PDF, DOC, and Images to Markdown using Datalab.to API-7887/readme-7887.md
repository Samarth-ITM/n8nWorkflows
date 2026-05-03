Convert PDF, DOC, and Images to Markdown using Datalab.to API

https://n8nworkflows.xyz/workflows/convert-pdf--doc--and-images-to-markdown-using-datalab-to-api-7887


# Convert PDF, DOC, and Images to Markdown using Datalab.to API

### 1. Workflow Overview

This workflow enables users to upload files (PDF, DOC, and images) via a web form and converts these files into Markdown format using the Datalab.to API. It is designed for use cases involving AI workflows that require document content in Markdown for easier processing, such as with large language models (LLMs).

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Accepts file uploads through a web form trigger.
- **1.2 File Conversion Request:** Sends the uploaded file to the Datalab.to API for conversion into Markdown.
- **1.3 Polling/Waiting:** Waits for the Datalab.to API to process the file.
- **1.4 Retrieval and Output Preparation:** Retrieves the converted Markdown content and prepares it for downstream use (e.g., further processing or storage).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures the user's file upload via a web form. It acts as the workflow’s entry point.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - Type: `Form Trigger`  
  - Role: Waits for a user to submit a form with a file upload.  
  - Configuration:  
    - Form titled "upload file" with a single required file input field labeled "file".  
    - Does not accept multiple files (single file only).  
    - Webhook ID configured to receive data when the form is submitted.  
  - Expressions/Variables: None beyond receiving the file input.  
  - Inputs: External HTTP form submission.  
  - Outputs: Passes the uploaded file data to the next node.  
  - Edge cases:  
    - User submits without a file (blocked by "required" field config).  
    - Large file uploads may cause timeouts depending on server limits.  
  - Version: 2.2

---

#### 2.2 File Conversion Request

**Overview:**  
This block sends the uploaded file to the Datalab.to API to convert it into Markdown format.

**Nodes Involved:**  
- Send to Datalab API

**Node Details:**  

- **Send to Datalab API**  
  - Type: `HTTP Request`  
  - Role: Submits the uploaded file as multipart form-data to the Datalab.to marker API endpoint for conversion.  
  - Configuration:  
    - POST request to `https://www.datalab.to/api/v1/marker`  
    - Content-Type: `multipart-form-data`  
    - Body parameters:  
      - `max_pages`: 4 (limits document pages processed)  
      - `use_llm`: true (enables use of large language model in processing)  
      - `output_format`: markdown (requests output in Markdown format)  
      - `file`: binary data from form input field "file"  
    - Authentication: HTTP Header Authentication using API key credential named "datalab.io" (header `X-API-Key`)  
  - Expressions/Variables: None dynamic; fixed parameters and binary file input.  
  - Inputs: Receives file from "On form submission" node.  
  - Outputs: Passes API response (likely containing a request_check_url or status) to the next node.  
  - Edge cases:  
    - API key invalid or missing → authentication error.  
    - File too large or unsupported format → API error or incomplete processing.  
    - Network timeout or API endpoint down.  
  - Version: 4.2  
  - Credential: Generic HTTP Header with key "X-API-Key", value your Datalab.to API key.

---

#### 2.3 Polling / Waiting

**Overview:**  
This block waits a fixed time before proceeding to allow the Datalab.to API to process the file asynchronously.

**Nodes Involved:**  
- Wait

**Node Details:**  

- **Wait**  
  - Type: `Wait`  
  - Role: Pauses the workflow for 10 seconds to allow file conversion to complete on the API side before polling for the results.  
  - Configuration: Wait duration = 10 seconds.  
  - Inputs: Receives response from "Send to Datalab API".  
  - Outputs: Triggers the next node which performs the GET request to retrieve the Markdown.  
  - Edge cases:  
    - Fixed wait may be insufficient for large files — may require longer wait or multiple retries (retry logic not implemented).  
  - Version: 1.1

---

#### 2.4 Retrieval and Output Preparation

**Overview:**  
This block retrieves the converted Markdown from the Datalab.to API and prepares it for further use.

**Nodes Involved:**  
- Get Markdown  
- Set Fields

**Node Details:**  

- **Get Markdown**  
  - Type: `HTTP Request`  
  - Role: Sends a GET request to the URL provided by the Datalab API to fetch the converted Markdown output.  
  - Configuration:  
    - URL is dynamic, obtained from the previous API response via expression `={{ $json.request_check_url }}`  
    - Authentication: Uses the same HTTP Header API key credential "datalab.io".  
  - Inputs: Triggered after the wait node.  
  - Outputs: Passes the retrieved Markdown content JSON to "Set Fields".  
  - Edge cases:  
    - If the file is not ready, this GET request may return incomplete or error status—retry logic is recommended but not implemented.  
    - Invalid or expired URL may cause failure.  
  - Version: 4.2

- **Set Fields**  
  - Type: `Set`  
  - Role: Extracts the Markdown content from the GET response and sets it into a dedicated field named `markdown`.  
  - Configuration:  
    - Sets `markdown` field with expression `={{ $json.markdown }}`  
  - Inputs: Receives JSON response from "Get Markdown".  
  - Outputs: Provides a clean data structure for downstream nodes or further processing.  
  - Edge cases:  
    - If `markdown` field is missing or empty, downstream nodes will receive empty content.  
  - Version: 3.4

---

### 3. Summary Table

| Node Name          | Node Type         | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                   |
|--------------------|-------------------|------------------------------------|-----------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger      | Accept user file upload            | (Webhook trigger)      | Send to Datalab API     | ## Convert Files to Markdown for LLM with Datalab.to                                                         |
| Send to Datalab API | HTTP Request      | Submit file for conversion         | On form submission     | Wait                    | ## Guide This simple automation enables you to convert .doc, .pdf, .png, .jpg, and .webp files to markdown... |
| Wait               | Wait              | Pause to allow processing          | Send to Datalab API    | Get Markdown            | ## Polling Confirmation Node You can add this node after "Get Markdown" node to confirm if the get request ...|
| Get Markdown       | HTTP Request      | Retrieve converted Markdown output | Wait                  | Set Fields              | ## Polling Confirmation Node You can add this node after "Get Markdown" node to confirm if the get request ...|
| Set Fields         | Set               | Extract & set markdown field       | Get Markdown           | (End)                   |                                                                                                               |
| Sticky Note13      | Sticky Note       | Visual diagram/image                |                       |                         | ![](https://res.cloudinary.com/dd6vlwblr/image/upload/v1756196495/Convert_PDF_DOC_IMAGES_to_1_sfdzu1.png)       |
| Sticky Note        | Sticky Note       | Title / description                 |                       |                         | ## Convert Files to Markdown for LLM with Datalab.to                                                          |
| Sticky Note2       | Sticky Note       | Guide & API key setup instructions |                       |                         | ## Guide Setup instructions and API reference link: [api reference](https://www.datalab.to/redoc#operation/marker_api_v1_marker_post)|
| Sticky Note3       | Sticky Note       | Polling confirmation recommendations|                       |                         | ## Polling Confirmation Node Advice on wait node and branching for success/failure                            |
| Sticky Note1       | Sticky Note       | Author contact info                 |                       |                         | ## Wanna work with me? Email: joseph@uppfy.com Twitter: [@juppfy](https://x.com/juppfy)                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node:**  
   - Type: Form Trigger  
   - Set form title: `upload file`  
   - Add one form field:  
     - Type: File  
     - Label: `file`  
     - Required: true  
     - Multiple files: false  
   - Save the webhook URL for form submission.

2. **Create "Send to Datalab API" HTTP Request node:**  
   - Method: POST  
   - URL: `https://www.datalab.to/api/v1/marker`  
   - Authentication: Generic HTTP Header Auth  
     - Credential Name: `datalab.io` (create with header key `X-API-Key` and your actual API key as value)  
   - Body Content Type: multipart-form-data  
   - Body Parameters:  
     - `max_pages`: `4` (string)  
     - `use_llm`: `true` (string)  
     - `output_format`: `markdown` (string)  
     - `file`: set as formBinaryData, inputDataFieldName: `file` (from previous node)  
   - Connect input from "On form submission".

3. **Create "Wait" node:**  
   - Set wait time: 10 seconds  
   - Connect input from "Send to Datalab API".

4. **Create "Get Markdown" HTTP Request node:**  
   - Method: GET  
   - URL: Use expression `={{ $json.request_check_url }}` to pull the URL from previous node output.  
   - Authentication: Use same `datalab.io` HTTP Header Auth.  
   - Connect input from "Wait".

5. **Create "Set Fields" node:**  
   - Add one assignment:  
     - Field name: `markdown`  
     - Value: expression `={{ $json.markdown }}`  
   - Connect input from "Get Markdown".

6. **Connect nodes in this sequence:**  
   - On form submission → Send to Datalab API → Wait → Get Markdown → Set Fields

7. (Optional) Add sticky notes for documentation and guidance, referencing the original notes for user instructions, API setup, polling logic, and contact info.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow converts .pdf, .doc, and common image files (.png, .jpg, .webp) to markdown, enabling LLM workflows to ingest document content easily.                  | Workflow purpose                                                                                            |
| API key setup: Create a Generic HTTP Header credential with header name `X-API-Key` and your Datalab.to API key as the value.                                         | Credential setup instructions                                                                               |
| For detailed API payload options and tweaks, see the official API reference at [Datalab.to API Reference](https://www.datalab.to/redoc#operation/marker_api_v1_marker_post) | Official API documentation                                                                                   |
| Polling confirmation: It’s recommended to implement a switch node (disabled in this workflow) to check the status of the conversion before proceeding or retrying.    | Workflow robustness and error handling advice                                                              |
| Contact: Joseph at joseph@uppfy.com, Twitter [@juppfy](https://x.com/juppfy)                                                                                          | Author contact                                                                                              |
| Visual overview image available: ![](https://res.cloudinary.com/dd6vlwblr/image/upload/v1756196495/Convert_PDF_DOC_IMAGES_to_1_sfdzu1.png)                             | Workflow architecture diagram                                                                               |

---

**Disclaimer:**  
The workflow content is generated exclusively from n8n automation, respecting all applicable content policies. It contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.