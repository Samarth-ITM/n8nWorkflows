Optimize Technical Manuals for RAG & Agents with Blockify IdeaBlocks

https://n8nworkflows.xyz/workflows/optimize-technical-manuals-for-rag---agents-with-blockify-ideablocks-9593


# Optimize Technical Manuals for RAG & Agents with Blockify IdeaBlocks

### 1. Workflow Overview

This workflow is designed to optimize technical manuals into structured, AI-friendly data blocks called "IdeaBlocks" for Retrieval-Augmented Generation (RAG) and agent-based applications. The workflow automates the process of extracting text from technical PDF manuals, chunking the content intelligently by headers, assembling contextual payloads, and then using a specialized API (Blockify) to convert these chunks into optimized XML-based IdeaBlocks. Finally, it aggregates and saves the output for further use, such as enhanced search or knowledge retrieval.

The workflow consists of the following major logical blocks:

- **1.1 Document Ingestion & Preparation:**  
  Search and download technical manual files (PDFs) from Google Drive, upload them to AWS S3, and generate signed URLs for processing.

- **1.2 PDF Text Extraction & Polling:**  
  Initiate a PDF extraction process via an external API, poll for completion, and download the extracted raw markdown text.

- **1.3 Markdown Parsing & Chunking:**  
  Parse the extracted markdown text, split it into meaningful chunks based on header levels, preparing it for AI processing.

- **1.4 Payload Assembly for Blockify API:**  
  Assemble each chunk with its preceding and following context into a structured payload format optimized for Blockify’s LLM ingestion.

- **1.5 IdeaBlock Generation via Blockify API:**  
  Send the payloads to the Blockify technical-ingest API, extract the XML IdeaBlock results, and aggregate them.

- **1.6 Output Post-Processing and Storage:**  
  Clean and merge the aggregated IdeaBlocks, convert to a text file, and upload the final optimized manual to Google Drive for storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Document Ingestion & Preparation

- **Overview:**  
  This block searches for source documents on Google Drive, downloads them, uploads files to an AWS S3 bucket, and generates signed URLs for secure access. These URLs are used to initiate PDF processing.

- **Nodes Involved:**  
  - Schedule Trigger2  
  - Search files and folders  
  - Loop over Documents to Blockify  
  - Download Document to Blockify  
  - Upload a Document to S3  
  - Get AWS Signed URL  
  - Initiate PDF Extraction

- **Node Details:**

  - **Schedule Trigger2**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a recurring schedule (interval set to run regularly).  
    - Configuration: Default interval (likely every minute or hourly depending on settings).  
    - Input: None  
    - Output: Triggers the search for files.

  - **Search files and folders**  
    - Type: Google Drive node (fileFolder resource)  
    - Role: Searches all files and folders within Google Drive root or specified folder.  
    - Configuration: Filter set to root folder; returns all files.  
    - Credentials: Google Drive OAuth2  
    - Input: Trigger from Schedule Trigger2  
    - Output: List of files found.

  - **Loop over Documents to Blockify**  
    - Type: SplitInBatches  
    - Role: Iterates over each found file to process sequentially or in batches.  
    - Configuration: Default batch size, no special options.  
    - Input: Files from search node  
    - Output: Passes each file to download step.

  - **Download Document to Blockify**  
    - Type: Google Drive node (download operation)  
    - Role: Downloads the binary content of each file by fileId.  
    - Credentials: Google Drive OAuth2  
    - Input: File item from Loop over Documents  
    - Output: Binary data of the file for upload.

  - **Upload a Document to S3**  
    - Type: AWS S3 node (upload operation)  
    - Role: Uploads the downloaded file binary to a specified AWS S3 bucket and folder path.  
    - Configuration: Bucket name and parent folder key specified.  
    - Credentials: AWS credentials  
    - Input: Downloaded file binary  
    - Output: Confirmation of upload.

  - **Get AWS Signed URL**  
    - Type: Code node  
    - Role: Generates a presigned URL for the uploaded S3 file to allow secure temporary access for processing.  
    - Configuration: Uses AWS credentials, bucket info, and key; signs URL valid for 8400 seconds (2h20m).  
    - Input: Upload confirmation and file info  
    - Output: JSON containing presigned URL and file metadata.  
    - Potential failures: Incorrect AWS credentials, missing fileName in binary data, permission issues.

  - **Initiate PDF Extraction**  
    - Type: HTTP Request  
    - Role: Calls an external API (Google Gemini PDF processing) to start asynchronous PDF to markdown extraction using the signed S3 URL.  
    - Configuration: POST request with API key header and signed URL in body.  
    - Input: Presigned URL from previous node  
    - Output: Job status info for polling.  
    - Edge cases: API key invalid, network errors, malformed URLs.

---

#### 1.2 PDF Text Extraction & Polling

- **Overview:**  
  Polls the external PDF extraction API until the extraction job status is "COMPLETED". Once done, it downloads the extracted raw markdown text.

- **Nodes Involved:**  
  - PDF Status Polling  
  - If Completed Poller  
  - Wait  
  - Download Final PDF Extracted Text Output  
  - Extract Raw Markdown from PDF Text

- **Node Details:**

  - **PDF Status Polling**  
    - Type: HTTP Request  
    - Role: Polls the PDF extraction status API with the job URL and API key header.  
    - Input: Job info from Initiate PDF Extraction  
    - Output: Current extraction status JSON.  
    - Failures: Timeout if job takes too long, API rate limits.

  - **If Completed Poller**  
    - Type: If  
    - Role: Checks if the extraction status equals "COMPLETED".  
    - Input: Status JSON from PDF Status Polling  
    - Output:  
      - True branch: proceeds to download extracted text  
      - False branch: loops back to continue polling.  
    - Edge cases: Status field missing or unexpected values.

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay between polling attempts to avoid overloading the API.  
    - Input: False branch from If Completed Poller  
    - Output: Loops back to PDF Status Polling.

  - **Download Final PDF Extracted Text Output**  
    - Type: HTTP Request  
    - Role: Downloads the final extracted markdown text once extraction is complete.  
    - Input: Completed job ID info  
    - Output: Extracted markdown text as raw data.  
    - Failures: Network issues, invalid job ID.

  - **Extract Raw Markdown from PDF Text**  
    - Type: Set node  
    - Role: Extracts and stores the raw markdown text into the workflow's JSON under the property "output".  
    - Input: Raw data from download node  
    - Output: JSON with markdown string.

---

#### 1.3 Markdown Parsing & Chunking

- **Overview:**  
  Parses the raw markdown text and splits it into manageable chunks based on Markdown header levels (H1, H2, H3). This hierarchical chunking is essential to preserve context and order for AI processing.

- **Nodes Involved:**  
  - Technical Manual Split Chunks

- **Node Details:**

  - **Technical Manual Split Chunks**  
    - Type: Code node  
    - Role: Implements custom JavaScript logic to split markdown text by headers (#, ##, ###) respecting fenced code blocks and limits chunk sizes (H1 ≤ 4000 chars, H2 ≤ 5000 chars).  
    - Configuration: Uses regex to detect headers and code fences, outputs ordered chunks with content and order index.  
    - Input: Markdown text from previous Set node  
    - Output: Array of chunk objects each with `chunk` (text) and `order` (sequence) fields.  
    - Edge cases: Markdown without headers, very large sections, nested code blocks.

---

#### 1.4 Payload Assembly for Blockify API

- **Overview:**  
  Constructs a structured payload for each chunk including the primary chunk and its immediate preceding and following chunks to provide contextual information for the Blockify API.

- **Nodes Involved:**  
  - Technical Manual Prompt Payload Assembly

- **Node Details:**

  - **Technical Manual Prompt Payload Assembly**  
    - Type: Code node  
    - Role: Sorts chunks by order and for each chunk creates a payload string with three labeled sections: Primary, Proceeding, and Following.  
    - Configuration: Template with section delimiters and chunk content; outputs JSON with payload and order metadata.  
    - Input: Chunks from split code node  
    - Output: Array of payload objects, each representing one chunk with context.  
    - Edge cases: Missing preceding or following chunks at edges (first/last chunk).

---

#### 1.5 IdeaBlock Generation via Blockify API

- **Overview:**  
  Sends each assembled payload to the Blockify technical-ingest API, receives XML IdeaBlock content, extracts the relevant text, and aggregates all results for final processing.

- **Nodes Involved:**  
  - Loop Over Chunks for Blockify  
  - Blockify Technical Ingest API  
  - Extract IdeaBlocks from API Response  
  - Aggregate all Manual Sections

- **Node Details:**

  - **Loop Over Chunks for Blockify**  
    - Type: SplitInBatches  
    - Role: Iterates over each payload to send individually to the Blockify API.  
    - Input: Payloads from assembly node  
    - Output: Passes each payload to API call node.

  - **Blockify Technical Ingest API**  
    - Type: HTTP Request  
    - Role: Calls Blockify API with bearer token authentication, sends payload as JSON in a chat completion format using the "technical-ingest" LLM model.  
    - Configuration: POST to https://api.blockify.ai/v1/chat/completions, max_tokens=8000, temperature=0.5.  
    - Credentials: HTTP Bearer token for Blockify API  
    - Input: Payload JSON from loop  
    - Output: API response with IdeaBlock XML.

  - **Extract IdeaBlocks from API Response**  
    - Type: Set node  
    - Role: Extracts the IdeaBlock XML content from the API response JSON into a simplified property "manual-section".  
    - Input: API response  
    - Output: JSON with manual-section containing XML content string.

  - **Aggregate all Manual Sections**  
    - Type: Aggregate node  
    - Role: Collects all manual-section outputs from each chunk into a single array "manual-sections" for further processing.  
    - Input: Multiple manual-section items from extract node  
    - Output: Aggregated array of IdeaBlocks.

---

#### 1.6 Output Post-Processing and Storage

- **Overview:**  
  Cleans the aggregated XML IdeaBlocks by stripping code fences and unwanted characters, converts the result into a text file, and uploads the final blockified manual to a designated Google Drive folder.

- **Nodes Involved:**  
  - Strip and Clean to Aggregate XML  
  - Convert to File  
  - Upload Blockified Manual

- **Node Details:**

  - **Strip and Clean to Aggregate XML**  
    - Type: Set node  
    - Role: Concatenates all manual-sections, extracts text content safely handling various possible JSON structures, and removes markdown code fences (``` etc).  
    - Input: Aggregated manual-sections array  
    - Output: Cleaned XML string under property "output".  
    - Edge cases: Variability in API response structure, empty or malformed content.

  - **Convert to File**  
    - Type: ConvertToFile node  
    - Role: Converts the cleaned XML string to a UTF-8 encoded text file with a dynamically generated file name based on the first headline of the content.  
    - Configuration: Filename derived by removing leading markdown heading markers and punctuation from the first non-empty line.  
    - Input: Cleaned XML string  
    - Output: Binary file content for upload.

  - **Upload Blockified Manual**  
    - Type: Google Drive node (upload operation)  
    - Role: Uploads the final blockified manual text file to a specific folder in Google Drive ("n8n-blockify-manual-extraction").  
    - Credentials: Google Drive OAuth2  
    - Input: File binary from convert node  
    - Output: Confirmation of upload  
    - Edge cases: Permissions, quota limits on Google Drive.

---

### 3. Summary Table

| Node Name                         | Node Type             | Functional Role                                | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                   |
|----------------------------------|-----------------------|-----------------------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger2                 | Schedule Trigger      | Starts workflow on schedule                    | -                               | Search files and folders           |                                                                                               |
| Search files and folders          | Google Drive          | Searches for source files                       | Schedule Trigger2               | Loop over Documents to Blockify    |                                                                                               |
| Loop over Documents to Blockify   | SplitInBatches        | Loops over each found document                  | Search files and folders        | Download Document to Blockify      |                                                                                               |
| Download Document to Blockify     | Google Drive          | Downloads document binary                        | Loop over Documents to Blockify | Upload a Document to S3            |                                                                                               |
| Upload a Document to S3           | AWS S3                | Uploads document to S3 bucket                   | Download Document to Blockify   | Get AWS Signed URL                 |                                                                                               |
| Get AWS Signed URL                | Code                  | Generates presigned S3 URL                       | Upload a Document to S3         | Initiate PDF Extraction            |                                                                                               |
| Initiate PDF Extraction           | HTTP Request          | Starts PDF extraction job via external API      | Get AWS Signed URL              | PDF Status Polling                 |                                                                                               |
| PDF Status Polling                | HTTP Request          | Polls status of PDF extraction job              | Initiate PDF Extraction / If Completed Poller (false branch) | Wait / If Completed Poller         |                                                                                               |
| If Completed Poller               | If                    | Checks if PDF extraction job is completed       | PDF Status Polling              | Download Final PDF Extracted Text Output / PDF Status Polling |                                                                                               |
| Wait                             | Wait                  | Waits between polling attempts                   | If Completed Poller (false)     | PDF Status Polling                  |                                                                                               |
| Download Final PDF Extracted Text Output | HTTP Request   | Downloads extracted markdown text                | If Completed Poller (true)      | Extract Raw Markdown from PDF Text |                                                                                               |
| Extract Raw Markdown from PDF Text| Set                   | Extracts raw markdown text                        | Download Final PDF Extracted Text Output | Technical Manual Split Chunks      |                                                                                               |
| Technical Manual Split Chunks    | Code                  | Splits markdown text into header-based chunks   | Extract Raw Markdown from PDF Text | Technical Manual Prompt Payload Assembly | Sticky Note3: Markdown chunking guidelines link included                                    |
| Technical Manual Prompt Payload Assembly | Code            | Assembles chunk payloads with context            | Technical Manual Split Chunks  | Loop Over Chunks for Blockify       | Sticky Note4: Payload format documentation                                                  |
| Loop Over Chunks for Blockify    | SplitInBatches        | Iterates over chunk payloads for API ingestion  | Technical Manual Prompt Payload Assembly | Aggregate all Manual Sections / Blockify Technical Ingest API |                                                                                               |
| Blockify Technical Ingest API    | HTTP Request          | Calls Blockify API to convert chunks to IdeaBlocks | Loop Over Chunks for Blockify   | Extract IdeaBlocks from API Response | Sticky Note5: Blockify API info and signup link                                            |
| Extract IdeaBlocks from API Response | Set                | Extracts IdeaBlock XML content from API response | Blockify Technical Ingest API  | Loop Over Chunks for Blockify       | Sticky Note6: Notes on output collection and alternative output methods                    |
| Aggregate all Manual Sections    | Aggregate             | Aggregates all IdeaBlock sections into array    | Extract IdeaBlocks from API Response | Strip and Clean to Aggregate XML   |                                                                                               |
| Strip and Clean to Aggregate XML | Set                   | Cleans and concatenates all IdeaBlocks into XML | Aggregate all Manual Sections  | Convert to File                    |                                                                                               |
| Convert to File                  | ConvertToFile          | Converts cleaned XML string to text file         | Strip and Clean to Aggregate XML | Upload Blockified Manual           |                                                                                               |
| Upload Blockified Manual         | Google Drive           | Uploads final blockified manual to Google Drive | Convert to File                | Loop over Documents to Blockify     |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run at desired interval (e.g., every hour or daily).

2. **Add a Google Drive node to search files**  
   - Operation: fileFolder -> Search  
   - Filter folder: root or specific folder ID  
   - Credentials: Google Drive OAuth2  
   - Connect Schedule Trigger → Search files and folders

3. **Add SplitInBatches node to loop over files**  
   - Default batch size (e.g., 1)  
   - Connect Search files and folders → Loop over Documents to Blockify

4. **Add Google Drive node to download each file**  
   - Operation: Download  
   - File ID: `={{ $json.id }}`  
   - Credentials: Google Drive OAuth2  
   - Connect Loop over Documents → Download Document to Blockify

5. **Add AWS S3 node to upload downloaded file**  
   - Operation: Upload  
   - Bucket: Your AWS bucket name  
   - Additional Fields: parentFolderKey (folder path in bucket)  
   - Credentials: AWS IAM user with S3 access  
   - Connect Download Document to Blockify → Upload a Document to S3

6. **Add Code node to generate presigned S3 URL**  
   - Paste provided JavaScript code that signs URLs with AWS v4 signature  
   - Set AWS keys, bucket, region, and key prefix accordingly  
   - Connect Upload a Document to S3 → Get AWS Signed URL

7. **Add HTTP Request node to initiate PDF extraction**  
   - Method: POST  
   - URL: Your PDF extraction API endpoint  
   - Headers: API key header set  
   - Body: JSON with signed_s3_url from previous node  
   - Connect Get AWS Signed URL → Initiate PDF Extraction

8. **Add HTTP Request node to poll PDF extraction status**  
   - URL: Use polling URL from Initiate PDF Extraction response  
   - Headers: API key header  
   - Connect Initiate PDF Extraction → PDF Status Polling

9. **Add If node to check if status == "COMPLETED"**  
   - Condition: `$json.status` equals "COMPLETED" (case sensitive)  
   - Connect PDF Status Polling → If Completed Poller

10. **Add Wait node for retry delay**  
    - No parameters (default wait) or set desired delay  
    - Connect If Completed Poller (false branch) → Wait → PDF Status Polling (loop)

11. **Add HTTP Request node to download final extracted text**  
    - URL: Extraction output endpoint including job ID  
    - Headers: API key header  
    - Connect If Completed Poller (true branch) → Download Final PDF Extracted Text Output

12. **Add Set node to extract raw markdown**  
    - Assign `output` field with the raw markdown text (`$json.data`)  
    - Connect Download Final PDF Extracted Text Output → Extract Raw Markdown from PDF Text

13. **Add Code node to split markdown into chunks**  
    - Paste JavaScript code that splits by headers, respecting code fences, chunk size limits  
    - Connect Extract Raw Markdown from PDF Text → Technical Manual Split Chunks

14. **Add Code node to assemble payloads**  
    - Paste JavaScript code assembling payloads with Primary, Proceeding, and Following sections  
    - Connect Technical Manual Split Chunks → Technical Manual Prompt Payload Assembly

15. **Add SplitInBatches node to loop over payloads**  
    - Default batch size  
    - Connect Technical Manual Prompt Payload Assembly → Loop Over Chunks for Blockify

16. **Add HTTP Request node to call Blockify API**  
    - Method: POST  
    - URL: https://api.blockify.ai/v1/chat/completions  
    - Authentication: HTTP Bearer Auth with Blockify token  
    - Body: JSON with `messages` array containing payload, model "technical-ingest", max_tokens 8000, temperature 0.5  
    - Connect Loop Over Chunks for Blockify → Blockify Technical Ingest API

17. **Add Set node to extract IdeaBlock XML content**  
    - Assign `manual-section` to the first choice message content from API response  
    - Connect Blockify Technical Ingest API → Extract IdeaBlocks from API Response

18. **Add Aggregate node to collect all manual sections**  
    - Aggregate all item JSON data into an array field `manual-sections`  
    - Connect Extract IdeaBlocks from API Response → Aggregate all Manual Sections

19. **Add Set node to clean and concatenate XML**  
    - Using expression, join all manual-section strings, strip markdown code fences  
    - Connect Aggregate all Manual Sections → Strip and Clean to Aggregate XML

20. **Add ConvertToFile node to convert cleaned XML string**  
    - Operation: toText  
    - Encoding: UTF-8  
    - File name generated dynamically from first headline in content, sanitized of punctuation  
    - Source property: `output` from previous node  
    - Connect Strip and Clean to Aggregate XML → Convert to File

21. **Add Google Drive node to upload final file**  
    - Operation: Upload  
    - Destination folder: e.g., "n8n-blockify-manual-extraction" folder ID  
    - Credentials: Google Drive OAuth2  
    - Connect Convert to File → Upload Blockified Manual

22. **Connect final output to Loop over Documents to Blockify (optional loop)**  
    - Allows processing multiple documents in sequence.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Blockify is a data optimization tool that converts unstructured technical manuals into structured IdeaBlocks, improving accuracy of LLMs by approximately 78x and reducing data size to 2.5% while preserving facts and numerical data.                                                                                                                                                                                                                                                                                                                                                                                             | See Sticky Note14 content in the workflow.                                                        |
| Markdown Document Chunking Guidelines are detailed in a Google Doc: https://docs.google.com/document/d/14goJMiMm1gU4XWe4mWWQN5ao8dKi5bwg5zrNEZR18oc/edit?tab=t.0#heading=h.82v51btyigaq                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note3                                                                                       |
| Blockify API free trial signup: https://console.blockify.ai/signup                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note5                                                                                       |
| Technical whitepaper and accuracy case studies are available at: https://iternal.ai/blockify and related links in Sticky Note14                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note14                                                                                      |
| Users may choose alternative output storage methods such as databases instead of Google Drive for IdeaBlock XML data storage and retrieval.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note6                                                                                       |
| Input PDF extraction uses proprietary Google Gemini API integration but can be replaced with any PDF-to-markdown extraction method as long as output markdown with headers is produced.                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note and workflow description overview                                                    |
| AWS S3 bucket and credentials must be configured correctly; the signed URL code node requires Node.js crypto module support.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | See Get AWS Signed URL node details                                                              |

---

This completes the comprehensive reference documentation for the "Optimize Technical Manuals for RAG & Agents with Blockify IdeaBlocks" n8n workflow.