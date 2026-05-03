Process Large Documents with OCR using SubworkflowAI and Gemini

https://n8nworkflows.xyz/workflows/process-large-documents-with-ocr-using-subworkflowai-and-gemini-10566


# Process Large Documents with OCR using SubworkflowAI and Gemini

### 1. Workflow Overview

This workflow automates the processing of large documents by leveraging the SubworkflowAI Extract API for OCR preparation and Google Gemini (PaLM) for document transcription using a multimodal language model (VLM). It is designed for use cases where documents are too large to be processed locally or in a single AI call, enabling efficient asynchronous handling, incremental data retrieval, and OCR transcription of document pages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Upload:**  
  Reception of the binary document file from Google Drive and uploading it to the SubworkflowAI Extract API for asynchronous processing.

- **1.2 Job Monitoring (Polling):**  
  Polling the SubworkflowAI job endpoint repeatedly until the asynchronous extraction job completes successfully or fails.

- **1.3 Dataset Retrieval:**  
  Once the job is complete, retrieving the dataset metadata and paginated dataset items (document pages) from SubworkflowAI.

- **1.4 Document OCR via Visual Language Model:**  
  Processing each retrieved page via Google Gemini’s multimodal model to transcribe images into Markdown text.

- **1.5 Workflow Control and Notes:**  
  Includes manual trigger, wait logic, and informative sticky notes to explain the workflow’s steps and resources.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Upload

**Overview:**  
This block downloads a document file from Google Drive and uploads it to SubworkflowAI’s Extract API to initiate asynchronous processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Download file (Google Drive)  
- Extract API (HTTP Request)  
- Sticky Note (Introductory note on Extract API usage)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually without external input  
  - Inputs: None  
  - Outputs: Triggers "Download file" node  
  - Failure: None expected, manual trigger only

- **Download file**  
  - Type: Google Drive Node  
  - Role: Downloads the document binary using a fixed file ID from Google Drive  
  - Configuration: Operation is "download", fileId is statically set to `"1wS9U7MQDthj57CvEcqG_Llkr-ek6RqGA"`  
  - Credentials: Google Drive OAuth2  
  - Inputs: Trigger from manual node  
  - Outputs: Sends binary file data to "Extract API" node  
  - Edge Cases:  
    - File not found or access denied by Google Drive (auth error)  
    - Network failures  
  - Version: v3

- **Extract API**  
  - Type: HTTP Request  
  - Role: Uploads the binary file to SubworkflowAI Extract API (multipart form-data)  
  - Configuration: POST to `https://api.subworkflow.ai/v1/extract`  
    - Body includes binary data field "file" and `expiresInDays` set to 0 (no expiry)  
  - Authentication: HTTP Header Auth using SubworkflowAI API Key  
  - Retry: Enabled with 5s interval between retries  
  - Inputs: Receives binary from "Download file"  
  - Outputs: Produces job object with job ID for polling  
  - Edge Cases:  
    - Upload failures, API auth errors, network timeout  
    - API rate limits or server errors  
  - Version: v4.2

- **Sticky Note**  
  - Provides documentation link and context on Extract API usage and asynchronous job firing

---

#### 1.2 Job Monitoring (Polling)

**Overview:**  
Polls the SubworkflowAI job status endpoint repeatedly until the job transitions to SUCCESS or ERROR, indicating completion or failure.

**Nodes Involved:**  
- Job Complete? (IF node)  
- Check Job Status (HTTP Request)  
- Wait (Delay)  
- Sticky Note1 (Job polling explanation)

**Node Details:**

- **Job Complete?**  
  - Type: IF Node  
  - Role: Examines job status returned from API to determine if job is done or still running  
  - Condition: Checks if `data.status` equals "SUCCESS" or "ERROR"  
  - Inputs: Receives job data from "Extract API" or "Check Job Status"  
  - Outputs:  
    - True branch proceeds to "Get Dataset"  
    - False branch loops to "Check Job Status" for polling  
  - Edge Cases:  
    - Unexpected status values or missing status field causing condition to fail

- **Check Job Status**  
  - Type: HTTP Request  
  - Role: Gets current job status from `https://api.subworkflow.ai/v1/jobs/{{jobId}}`  
  - Authentication: SubworkflowAI API Key  
  - Inputs: Looped from "Job Complete?" false branch  
  - Outputs: Job status JSON to "Wait" node  
  - Edge Cases:  
    - API auth failure, network errors, invalid job ID

- **Wait**  
  - Type: Wait Node  
  - Role: Delays 1 second before next poll to avoid hammering the API  
  - Inputs: From "Check Job Status"  
  - Outputs: Back to "Job Complete?" to continue polling loop  
  - Edge Cases: None

- **Sticky Note1**  
  - Explains the job polling mechanism and references official Job API docs

---

#### 1.3 Dataset Retrieval

**Overview:**  
After job completion, retrieves dataset metadata and paginated dataset items (document pages with images) from SubworkflowAI.

**Nodes Involved:**  
- Get Dataset (HTTP Request)  
- Get Dataset Items (HTTP Request with Pagination)  
- Split Out (Split Out node)  
- Sticky Note2 (Dataset retrieval explanation)  
- Sticky Note4 (Use-case for document OCR with Gemini)

**Node Details:**

- **Get Dataset**  
  - Type: HTTP Request  
  - Role: Retrieves metadata of the dataset identified by `datasetId` from job result  
  - URL: `https://api.subworkflow.ai/v1/datasets/{{ datasetId }}`  
  - Authentication: SubworkflowAI API Key  
  - Inputs: From "Job Complete?" node true branch  
  - Outputs: Dataset metadata to "Get Dataset Items"  
  - Edge Cases:  
    - Dataset not found, API auth errors

- **Get Dataset Items**  
  - Type: HTTP Request  
  - Role: Retrieves dataset items (pages) in batches of 10 with pagination support  
  - URL: `https://api.subworkflow.ai/v1/datasets/{{ datasetId }}/items?row=jpg&limit=10`  
  - Pagination: Offset-based, increments offset by 10 up to 5 pages max, breaks early if fewer than 10 items returned  
  - Authentication: SubworkflowAI API Key  
  - Inputs: From "Get Dataset"  
  - Outputs: Dataset items array to "Split Out" node  
  - Edge Cases:  
    - Pagination errors, incomplete data, API limits

- **Split Out**  
  - Type: Split Out  
  - Role: Splits dataset items array into individual items (pages) for further processing  
  - Field to split: "data"  
  - Inputs: From "Get Dataset Items"  
  - Outputs: Single dataset item per execution to "Document OCR via VLM"  
  - Edge Cases: Empty or malformed data arrays

- **Sticky Note2**  
  - Provides links and explanation for Datasets API usage and rationale for incremental retrieval

- **Sticky Note4**  
  - Describes the VLM OCR use case and benefits of using public share links for efficient image processing

---

#### 1.4 Document OCR via Visual Language Model

**Overview:**  
Transcribes each dataset page image to Markdown text using Google Gemini (PaLM) multimodal AI model.

**Nodes Involved:**  
- Document OCR via VLM (Google Gemini node)  
- Sticky Note6 (Next steps note)

**Node Details:**

- **Document OCR via VLM**  
  - Type: Google Gemini (PaLM) Node (Langchain integration)  
  - Role: Sends each page’s public share URL to Gemini to transcribe image content into Markdown  
  - Parameters:  
    - Text prompt: "Transcribe this image to Markdown"  
    - Model: "models/gemini-2.5-flash"  
    - Resource: Image URLs from `{{ $json.share.url }}` field in dataset item  
    - Operation: Analyze (image analysis)  
  - Credentials: Google Palm API Key (OAuth2 or API Key)  
  - Inputs: Single dataset item (page) from "Split Out" node  
  - Outputs: Transcription result (Markdown)  
  - Edge Cases:  
    - Invalid or expired share URLs  
    - API quota exceeded or auth problems  
    - Model downtime or response errors

- **Sticky Note6**  
  - Points to next steps with the Search API for post-processing or querying results

---

#### 1.5 Workflow Control and Notes

**Overview:**  
Contains manual trigger, wait node, and several sticky notes that provide documentation, instructions, and branding information to assist developers.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Wait  
- Multiple Sticky Notes (Introductory, polling explanation, dataset explanation, use case, and branding)

**Node Details:**

- Sticky notes contain Markdown formatted text with helpful links to SubworkflowAI and n8n documentation, usage instructions, and a banner for Subworkflow.ai with community links.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                              | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                                                                                                                                             |
|-------------------------|--------------------------------|----------------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Workflow entry point                          | None                         | Download file                |                                                                                                                                                                                                                                        |
| Download file           | Google Drive                   | Download document binary file                  | When clicking ‘Execute workflow’ | Extract API                  |                                                                                                                                                                                                                                        |
| Extract API             | HTTP Request                  | Upload binary file to SubworkflowAI Extract API | Download file                 | Job Complete?                | ### 1. Upload Binary File to Extract API [Extract API Documentation](https://docs.subworkflow.ai/api-reference/post-v1-extract) Our workflow starts with uploading our document to the SubworkflowAI service.                          |
| Job Complete?           | IF                            | Check if extract job is complete               | Extract API, Wait            | Get Dataset (true), Check Job Status (false) | ### 2. Poll for Async "Extract" Job to Complete [Jobs API Documentation](https://docs.subworkflow.ai/api-reference/get-v1-jobs-id) Poll job status until SUCCESS or ERROR.                                                             |
| Check Job Status        | HTTP Request                  | Poll job status from SubworkflowAI              | Job Complete? (false)        | Wait                         |                                                                                                                                                                                                                                        |
| Wait                    | Wait                          | Delay between job status polls                   | Check Job Status             | Job Complete?                |                                                                                                                                                                                                                                        |
| Get Dataset             | HTTP Request                  | Retrieve dataset metadata                       | Job Complete? (true)         | Get Dataset Items            | ### 3. Fetch Resulting Dataset and Get Dataset Items [Datasets API Documentation](https://docs.subworkflow.ai/api-reference/get-v1-datasets)                                                                                           |
| Get Dataset Items       | HTTP Request                  | Retrieve dataset pages/items with pagination    | Get Dataset                  | Split Out                   |                                                                                                                                                                                                                                        |
| Split Out               | Split Out                     | Split dataset items array into single items     | Get Dataset Items            | Document OCR via VLM         |                                                                                                                                                                                                                                        |
| Document OCR via VLM    | Google Gemini (PaLM)          | Transcribe each page image via multimodal LLM   | Split Out                   |                              | ### 4. Example VLM Use-Case - Document OCR [Learn more about the Gemini node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini/) Benefits include no additional binary passing between nodes. |
| Sticky Note             | Sticky Note                   | Introductory/explanatory text                    | None                        | None                        | See detailed notes in section 2.                                                                                                                                                                                                       |
| Sticky Note1            | Sticky Note                   | Polling explanation and job API docs             | None                        | None                        |                                                                                                                                                                                                                                        |
| Sticky Note2            | Sticky Note                   | Dataset retrieval explanation                      | None                        | None                        |                                                                                                                                                                                                                                        |
| Sticky Note4            | Sticky Note                   | VLM OCR use case explanation                       | None                        | None                        |                                                                                                                                                                                                                                        |
| Sticky Note6            | Sticky Note                   | Next steps with Search API                         | None                        | None                        |                                                                                                                                                                                                                                        |
| Sticky Note7            | Sticky Note                   | Branding, detailed instructions, prerequisites    | None                        | None                        | [Subworkflow.ai Banner and Documentation](https://subworkflow.ai?utm=n8n)                                                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters required. This node starts the workflow manually.

2. **Add Google Drive Node:**  
   - Name: `Download file`  
   - Operation: `download`  
   - File ID: Set to `"1wS9U7MQDthj57CvEcqG_Llkr-ek6RqGA"` (or your own file ID)  
   - Credentials: Configure Google Drive OAuth2 account with appropriate permissions.

3. **Add HTTP Request Node for Extract API Upload:**  
   - Name: `Extract API`  
   - Method: POST  
   - URL: `https://api.subworkflow.ai/v1/extract`  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `file`: set as form binary data from the incoming binary data field (e.g., "data")  
     - `expiresInDays`: set to 0 (no expiration)  
   - Authentication: HTTP Header Auth with SubworkflowAI API Key credential  
   - Enable retry on failure with 5000 ms wait between tries.

4. **Add IF Node to Check Job Completion:**  
   - Name: `Job Complete?`  
   - Condition: Check if `{{$json.data.status}}` equals "SUCCESS" OR "ERROR"  
   - True branch: proceed to dataset retrieval  
   - False branch: proceed to job status polling

5. **Add HTTP Request Node to Poll Job Status:**  
   - Name: `Check Job Status`  
   - Method: GET  
   - URL: `https://api.subworkflow.ai/v1/jobs/{{ $json.data.id }}`  
   - Authentication: SubworkflowAI API Key

6. **Add Wait Node:**  
   - Name: `Wait`  
   - Delay: 1 second (to avoid API flooding)

7. **Connect Polling Loop:**  
   - False output of `Job Complete?` → `Check Job Status` → `Wait` → back to `Job Complete?`

8. **Add HTTP Request Node to Get Dataset:**  
   - Name: `Get Dataset`  
   - Method: GET  
   - URL: `https://api.subworkflow.ai/v1/datasets/{{ $json.data.datasetId }}`  
   - Authentication: SubworkflowAI API Key  
   - Connect True output of `Job Complete?` to here

9. **Add HTTP Request Node to Get Dataset Items:**  
   - Name: `Get Dataset Items`  
   - Method: GET  
   - URL: `https://api.subworkflow.ai/v1/datasets/{{ $json.data.id }}/items?row=jpg&limit=10`  
   - Authentication: SubworkflowAI API Key  
   - Enable pagination:  
     - Pagination type: Offset  
     - Offset parameter name: `offset`  
     - Offset increment: 10  
     - Limit pages fetched: 5 pages max  
     - Stop when returned data length < 10  
     - Request interval: 500 ms  
   - Connect output of `Get Dataset` to this node

10. **Add Split Out Node:**  
    - Name: `Split Out`  
    - Field to split out: `data` (the array of dataset items)  
    - Connect output of `Get Dataset Items` to this node

11. **Add Google Gemini (PaLM) Node for OCR:**  
    - Name: `Document OCR via VLM`  
    - Resource: Image  
    - Operation: Analyze  
    - Model ID: `models/gemini-2.5-flash`  
    - Text prompt: `"Transcribe this image to Markdown"`  
    - Image URLs: Set to `{{$json.share.url}}` (public share link from dataset item)  
    - Credentials: Configure Google Palm API Key/OAuth2 credentials  
    - Connect output of `Split Out` to this node

12. **Add Sticky Notes as needed:**  
    - Add explanatory sticky notes at relevant points, referencing official API docs and usage instructions for SubworkflowAI and Gemini nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Subworkflow.ai enables processing of large documents (100 MB+ or up to 5000 pages) via asynchronous extraction jobs and dataset APIs, avoiding local memory overload.                                                               | https://subworkflow.ai                                                                                      |
| Extract API Documentation: API to upload files and start async jobs for document processing.                                                                                                                                         | https://docs.subworkflow.ai/api-reference/post-v1-extract                                                  |
| Jobs API Documentation: Poll job status to determine completion or failure.                                                                                                                                                          | https://docs.subworkflow.ai/api-reference/get-v1-jobs-id                                                    |
| Datasets API Documentation: Retrieve metadata and paginated pages of dataset for incremental processing.                                                                                                                            | https://docs.subworkflow.ai/api-reference/get-v1-datasets                                                   |
| Gemini node documentation for image transcription using Google Gemini (PaLM) multimodal models in n8n.                                                                                                                               | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini/                         |
| Join Subworkflow.ai Discord community for support and updates.                                                                                                                                                                       | https://discord.gg/RCHeCPJnYw                                                                               |
| Next suggested step after OCR is to utilize SubworkflowAI Search API for querying extracted data.                                                                                                                                     | https://docs.subworkflow.ai/api-reference/post-v1-search                                                    |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a no-code automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.