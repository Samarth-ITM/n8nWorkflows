PDF to Markdown Converter with LlamaCloud Parser

https://n8nworkflows.xyz/workflows/pdf-to-markdown-converter-with-llamacloud-parser-11811


# PDF to Markdown Converter with LlamaCloud Parser

### 1. Workflow Overview

This workflow, titled **PDF to Markdown Converter with LlamaCloud Parser**, automates the extraction and conversion of PDF documents into clean markdown format using the LlamaCloud parsing API. It is designed to handle complex PDF files including tables, images, and multi-column layouts, providing structured markdown output ready for further AI processing or content transformation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Retrieve the PDF file, primarily from Google Drive, but extensible to other sources.
- **1.2 Upload and Job Submission**: Upload the PDF binary data to LlamaCloud’s parsing API, initiating a parsing job.
- **1.3 Polling and Status Check Loop**: Wait and periodically check the job status until parsing completes.
- **1.4 Retrieval of Parsed Markdown**: Once parsing is successful, retrieve the markdown formatted output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block retrieves the PDF file to be parsed. The default source is Google Drive, where the file is downloaded as binary data. Alternative input nodes (HTTP Request, Binary File, Webhook) can replace the Google Drive node for other sources.

**Nodes Involved:**  
- Download File From Drive1

**Node Details:**

- **Download File From Drive1**  
  - Type: Google Drive node  
  - Role: Downloads a PDF file from Google Drive using the specified File ID.  
  - Configuration:
    - Operation: Download
    - File ID: Hardcoded to `1TPAxQq0fXVrgr7VbP7_RVV33v6r0sYk9` (example PDF file)
    - Credentials: Google Drive OAuth2 API (named "WFANMain")  
  - Input: None (start node)  
  - Output: Binary data of the downloaded PDF passed to the next node  
  - Potential Failures:
    - Authentication errors due to invalid or expired OAuth2 credentials.
    - Incorrect or inaccessible File ID.
    - Network timeouts or API rate limits from Google Drive.

---

#### 2.2 Upload and Job Submission

**Overview:**  
This block uploads the PDF binary data to LlamaCloud's parsing API and receives a job ID to track the parsing process.

**Nodes Involved:**  
- Send Data To Llama Cloud1

**Node Details:**

- **Send Data To Llama Cloud1**  
  - Type: HTTP Request node  
  - Role: Uploads the PDF file to LlamaCloud API to start parsing.  
  - Configuration:
    - Method: POST  
    - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`  
    - Content Type: multipart-form-data  
    - Body Parameters: Binary data field named "file" sourced from the PDF binary  
    - Authentication: Generic HTTP Header Auth with Bearer token (LlamaCloud API key)  
    - HTTP Header: Accept: application/json  
  - Input: Binary PDF data from "Download File From Drive1"  
  - Output: JSON response containing a job ID for subsequent status tracking  
  - Potential Failures:
    - Authentication failure if API key is invalid or missing.
    - Network errors or API downtime.
    - Invalid file format or corrupted binary data.
    - Exceeding API request limits.

---

#### 2.3 Polling and Status Check Loop

**Overview:**  
This block manages asynchronous polling by waiting and checking the job status repeatedly until the parsing job completes successfully.

**Nodes Involved:**  
- Wait1  
- Check Status1  
- Check Job Status1  
- Wait3

**Node Details:**

- **Wait1**  
  - Type: Wait node  
  - Role: Initial short wait (1 second) after submitting the job before first status check.  
  - Configuration: Wait 1 second  
  - Input: Output from "Send Data To Llama Cloud1"  
  - Output: Triggers "Check Status1"  
  - Potential Failures: Minimal; mostly timing-related issues.

- **Check Status1**  
  - Type: HTTP Request node  
  - Role: Queries LlamaCloud API for parsing job status using the job ID.  
  - Configuration:
    - Method: GET  
    - URL: Constructed dynamically as `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`  
    - Headers: Accept: application/json  
    - Authentication: Generic HTTP Header Auth with Bearer token (LlamaCloud API key)  
  - Input: Job ID from previous node  
  - Output: JSON containing job status (e.g., PENDING, SUCCESS)  
  - Potential Failures:
    - Authentication errors.
    - Invalid or expired job ID.
    - API rate limits.
    - Network failures.

- **Check Job Status1**  
  - Type: If node  
  - Role: Branching logic based on job status returned by "Check Status1".  
  - Configuration:
    - Condition: Checks if `$json.status` equals "SUCCESS".  
  - Input: JSON job status from "Check Status1"  
  - Output:  
    - If TRUE: Passes flow to "Get Data1" node to retrieve markdown.  
    - If FALSE: Passes flow to "Wait3" node to wait before next status check.  
  - Potential Edge Cases:
    - Unexpected status values.
    - Expression evaluation errors if `$json.status` is missing.

- **Wait3**  
  - Type: Wait node  
  - Role: Waits 30 seconds before the next status check attempt if job is still pending.  
  - Configuration: Wait 30 seconds  
  - Input: Output from "Check Job Status1" (FALSE branch)  
  - Output: Loops back to "Check Status1" for repeated polling  
  - Potential Failures: Timing issues or workflow execution timeouts if the job runs too long.

---

#### 2.4 Retrieval of Parsed Markdown

**Overview:**  
Once the parsing job completes successfully, this block fetches the final markdown content from LlamaCloud.

**Nodes Involved:**  
- Get Data1

**Node Details:**

- **Get Data1**  
  - Type: HTTP Request node  
  - Role: Retrieves the parsed markdown output for the completed job.  
  - Configuration:
    - Method: GET  
    - URL: Dynamically constructed as `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`  
    - Headers: Accept: application/json  
    - Authentication: Generic HTTP Header Auth with Bearer token (LlamaCloud API key)  
  - Input: Job ID from "Check Job Status1" (TRUE branch)  
  - Output: JSON containing clean markdown text with extracted content, tables, images, and formatting  
  - Potential Failures:
    - Authentication errors.
    - Job ID missing or invalid.
    - API errors or malformed response.
    - Network issues.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                      | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                   |
|---------------------------|-------------------------|------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow Info & Setup      | Sticky Note             | Documentation and setup instructions| -                      | -                       | Describes workflow purpose, setup steps for LlamaCloud API key, Google Drive, and execution notes.            |
| Step 1                    | Sticky Note             | Explains input PDF source options  | -                      | -                       | Details PDF source options: Google Drive, HTTP Request, Binary File, Webhook.                                |
| Step 2                    | Sticky Note             | Explains upload to LlamaCloud      | -                      | -                       | Describes sending PDF to LlamaCloud and obtaining Job ID; mentions API key credential requirements.          |
| Step 3                    | Sticky Note             | Explains waiting and polling logic | -                      | -                       | Details wait and check loop until parsing job status is SUCCESS.                                             |
| Step 4                    | Sticky Note             | Explains markdown retrieval         | -                      | -                       | Describes retrieval of final markdown output and its structure.                                              |
| Download File From Drive1  | Google Drive            | Downloads PDF from Google Drive    | -                      | Send Data To Llama Cloud1|                                                                                                               |
| Send Data To Llama Cloud1  | HTTP Request            | Uploads PDF to LlamaCloud API      | Download File From Drive1| Wait1                   | Requires Bearer token credential for LlamaCloud.                                                             |
| Wait1                     | Wait                    | Initial wait before status check   | Send Data To Llama Cloud1| Check Status1           |                                                                                                               |
| Check Status1             | HTTP Request            | Checks parsing job status          | Wait1                   | Check Job Status1        | Requires Bearer token credential for LlamaCloud.                                                             |
| Check Job Status1         | If                      | Branches on job completion status  | Check Status1           | Get Data1 / Wait3        |                                                                                                               |
| Wait3                     | Wait                    | Waits 30 seconds before retry      | Check Job Status1       | Check Status1            |                                                                                                               |
| Get Data1                 | HTTP Request            | Retrieves parsed markdown result   | Check Job Status1       | -                       | Requires Bearer token credential for LlamaCloud.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Google Drive node**:  
   - Name: `Download File From Drive1`  
   - Operation: `Download`  
   - Configure File ID to the desired PDF file (example: `1TPAxQq0fXVrgr7VbP7_RVV33v6r0sYk9`)  
   - Set up and assign Google Drive OAuth2 credentials.  
   - This node outputs the binary PDF file.

3. **Add an HTTP Request node** to upload the PDF to LlamaCloud:  
   - Name: `Send Data To Llama Cloud1`  
   - HTTP Method: `POST`  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`  
   - Authentication: Set up Generic Header Auth credential with header:  
     - Name: `Authorization`  
     - Value: `Bearer YOUR_LLAMA_API_KEY`  
   - Content Type: `multipart-form-data`  
   - Body Parameters: Add a parameter with:  
     - Name: `file`  
     - Type: `formBinaryData`  
     - Input Data Field Name: `data` (this binds to the binary data from Google Drive node)  
   - Headers: Add `Accept: application/json`  
   - Connect output of Google Drive node to this node.

4. **Add a Wait node**:  
   - Name: `Wait1`  
   - Wait time: 1 second  
   - Connect output of HTTP Request upload node to this node.

5. **Add an HTTP Request node** to check job status:  
   - Name: `Check Status1`  
   - HTTP Method: `GET`  
   - URL: Use expression: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}` where `$json.id` is the job ID from the upload response.  
   - Authentication: Use the same Generic Header Auth credential as upload node.  
   - Headers: Add `Accept: application/json`  
   - Connect output of `Wait1` to this node.

6. **Add an If node** to evaluate job status:  
   - Name: `Check Job Status1`  
   - Condition: `$json.status` equals `SUCCESS` (case-sensitive strict string compare)  
   - Connect output of `Check Status1` to this node.

7. **Add an HTTP Request node** to get the parsed markdown:  
   - Name: `Get Data1`  
   - HTTP Method: `GET`  
   - URL: Expression: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`  
   - Authentication: Same Generic Header Auth credential  
   - Headers: `Accept: application/json`  
   - Connect TRUE output of `Check Job Status1` to this node.

8. **Add a Wait node** for retry delay:  
   - Name: `Wait3`  
   - Wait time: 30 seconds  
   - Connect FALSE output of `Check Job Status1` to this node.

9. **Connect the output of `Wait3` back to `Check Status1`** to form the polling loop.

10. **Ensure all HTTP Request nodes calling LlamaCloud use the same Generic Header Auth credential** containing the Bearer token with your LlamaCloud API key.

11. **(Optional) Add sticky notes** to document each step and setup instructions for clarity.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow uses LlamaCloud API for parsing PDFs into markdown, which supports complex layouts including tables and images. | https://cloud.llamaindex.ai |
| To obtain the LlamaCloud API key, sign up or log in at LlamaCloud and create API keys under your account settings. | https://cloud.llamaindex.ai |
| Google Drive OAuth2 credentials require setup in n8n for file access; alternatively, replace Google Drive node with other input methods. | n8n.io/docs |
| The workflow implements automatic retry polling every 30 seconds until the parsing job completes successfully. | Internal workflow logic |
| Output markdown is suitable for AI analysis or further content transformations. | Workflow description notes |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.