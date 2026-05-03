Extract text from PDF and image using Vertex AI (Gemini) into CSV

https://n8nworkflows.xyz/workflows/extract-text-from-pdf-and-image-using-vertex-ai--gemini--into-csv-2614


# Extract text from PDF and image using Vertex AI (Gemini) into CSV

### 1. Workflow Overview

This workflow automates the extraction of textual data from PDF bank statements and image screenshots stored in Google Drive, processes this data using Vertex AI (Gemini) and OpenRouter AI language models, and outputs the extracted transactions as CSV files back into Google Drive for easy import into budgeting software.

It is designed for users who want to avoid manual data entry of digital expense transactions by automating text extraction and classification into categorized CSV data.

**Logical Blocks:**

- **1.1 Input Reception:** Triggered by file creation in a specific Google Drive folder where PDFs or images are uploaded.
- **1.2 File Type Routing:** Distinguishes uploaded files between PDFs and images to apply appropriate processing.
- **1.3 File Download:** Downloads the uploaded file from Google Drive for local processing.
- **1.4 PDF Text Extraction & AI Processing:** Extracts raw text from PDFs, sends it to OpenRouter AI for parsing and CSV conversion with categorization.
- **1.5 Image Text Extraction with Vertex AI:** Sends images to Vertex AI Gemini for text extraction and CSV conversion with categorization.
- **1.6 CSV Conversion and Upload:** Converts AI output to CSV files and uploads them to a designated Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Watches a specified Google Drive folder for newly created PDF or image files uploaded by the user.

- **Nodes Involved:**  
  - Get PDF or Images

- **Node Details:**  
  - **Get PDF or Images**  
    - Type: Google Drive Trigger  
    - Role: Initiates workflow when a file is created in a specific folder  
    - Configuration:  
      - Watches folder with ID `1HOeRP5iwccg93UPUYmWYD7DyDmRREkhj` ("Actual Budget")  
      - Poll interval: every minute  
      - Authentication: Google Service Account with appropriate Drive access  
    - Inputs: Google Drive folder event  
    - Outputs: File metadata including file id and mimeType  
    - Edge cases:  
      - Missing or incorrect folder permissions can block trigger  
      - Service account must have access to the folder  
      - Delay due to polling interval  
    - Version: 1

---

#### 1.2 File Type Routing

- **Overview:**  
  Checks MIME type of uploaded file to route the workflow to PDF or image processing branches.

- **Nodes Involved:**  
  - Route based on PDF or Image

- **Node Details:**  
  - **Route based on PDF or Image**  
    - Type: Switch  
    - Role: Branches workflow based on MIME type  
    - Configuration:  
      - Routes to "pdf" if MIME type equals `application/pdf`  
      - Routes to "image" if MIME type contains `image/`  
      - Uses expression `{{$json.mimeType}}`  
    - Inputs: File metadata from "Get PDF or Images"  
    - Outputs: Two branches leading to PDF or image download nodes  
    - Edge cases:  
      - Files with unsupported MIME types will not be processed  
      - MIME type detection errors could misroute files  
    - Version: 2

---

#### 1.3 File Download

- **Overview:**  
  Downloads the file from Google Drive to prepare it for extraction.

- **Nodes Involved:**  
  - Download PDF  
  - Download Image

- **Node Details:**  
  - **Download PDF**  
    - Type: Google Drive  
    - Role: Downloads PDF files using file ID  
    - Configuration:  
      - Operation: download  
      - File ID from `Get PDF or Images` node  
      - Auth: Google Service Account  
    - Inputs: File metadata from routing node  
    - Outputs: Binary PDF file  
    - Edge cases:  
      - File missing or permission denied errors  
      - Network timeouts  
    - Version: 3

  - **Download Image**  
    - Type: Google Drive  
    - Role: Downloads image files similarly to PDF  
    - Configuration: Similar to Download PDF  
    - Additional: `alwaysOutputData` enabled to force output even if empty  
    - Edge cases & version: Same as Download PDF

---

#### 1.4 PDF Text Extraction & AI Processing

- **Overview:**  
  Extracts raw text from the PDF, sends the text to OpenRouter AI to parse transactions, categorize them, and format as CSV.

- **Nodes Involved:**  
  - Extract data from PDF  
  - Send data to A.I.  
  - Convert to CSV  
  - Upload to Google Drive

- **Node Details:**  
  - **Extract data from PDF**  
    - Type: Extract From File  
    - Role: Extracts raw text from PDF binary data  
    - Configuration: Operation set to `pdf`  
    - Inputs: Binary PDF from "Download PDF"  
    - Outputs: Extracted text in JSON  
    - Edge cases:  
      - PDFs with complex layout or scanned images may yield incomplete text  
    - Version: 1

  - **Send data to A.I.**  
    - Type: HTTP Request  
    - Role: Sends extracted PDF text to OpenRouter AI for text analysis & CSV conversion  
    - Configuration:  
      - POST to `https://openrouter.ai/api/v1/chat/completions`  
      - Model: `meta-llama/llama-3.1-70b-instruct:free`  
      - Message prompt instructs AI to read bank statement text and export transactions as CSV with category column  
      - Authentication: HTTP Header Auth with Bearer token  
    - Inputs: Extracted text JSON from previous node  
    - Outputs: AI-generated CSV text  
    - Edge cases:  
      - API rate limits or auth failure  
      - Model response errors or timeouts  
      - Prompt encoding failure if text is malformed  
    - Version: 4.2

  - **Convert to CSV**  
    - Type: Convert To File  
    - Role: Converts AI response text to CSV file format  
    - Configuration: Default options  
    - Inputs: AI response JSON  
    - Outputs: CSV binary file  
    - Edge cases: Malformed AI output could cause conversion errors  
    - Version: 1.1

  - **Upload to Google Drive**  
    - Type: Google Drive  
    - Role: Uploads CSV file to a specified Drive folder  
    - Configuration:  
      - Folder ID: `1Zo4OFCv1qWRX1jo0VL_iqUBf4v0fZEXe` ("CSV Exports")  
      - Filename: Today's date (dynamic)  
      - Auth: Google Service Account  
    - Inputs: CSV binary file  
    - Outputs: Confirmation of upload  
    - Edge cases:  
      - Permission issues on folder  
      - Filename conflicts (date only naming may overwrite)  
    - Version: 3

---

#### 1.5 Image Text Extraction with Vertex AI

- **Overview:**  
  Passes downloaded images to Vertex AI Gemini model to extract transaction data as CSV with categorized columns.

- **Nodes Involved:**  
  - Vertex A.I. extract text  
  - Convert to CSV2  
  - Upload to Google Drive1

- **Node Details:**  
  - **Vertex A.I. extract text**  
    - Type: chainLlm (Langchain Google Gemini model)  
    - Role: Sends image binary to Vertex AI Gemini with prompt to extract transactions as CSV with categories  
    - Configuration:  
      - Prompt instructs to read image and export transaction CSV with category column, only CSV data with header row returned  
      - Uses HumanMessagePromptTemplate for image binary input  
    - Inputs: Binary image from "Download Image"  
    - Outputs: AI-generated CSV text  
    - Edge cases:  
      - Requires Vertex AI API enabled and correct IAM permissions on Google Cloud account  
      - Possible API errors or quota limits  
      - Complex images may reduce extraction accuracy  
    - Version: 1.4

  - **Convert to CSV2**  
    - Type: Convert To File  
    - Role: Converts AI response text to CSV file format (second instance)  
    - Inputs: AI response JSON from Vertex AI node  
    - Outputs: CSV binary file  
    - Edge cases: Same as Convert to CSV  
    - Version: 1.1

  - **Upload to Google Drive1**  
    - Type: Google Drive  
    - Role: Uploads CSV file to the same "CSV Exports" folder as PDF branch  
    - Configuration: Same as "Upload to Google Drive" node  
    - Inputs: CSV binary file  
    - Outputs: Confirmation of upload  
    - Edge cases: Same as above  
    - Version: 3

---

#### 1.6 Sticky Notes (Documentation)

- **Overview:**  
  Provides embedded documentation and instructions within the workflow for setup and usage.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  
  - **Sticky Note**  
    - Content: Overview and link to detailed blog post about the workflow  
  - **Sticky Note1**  
    - Content: Instructions for Google Drive folder setup and service account sharing  
  - **Sticky Note2**  
    - Content: Instructions for OpenRouter API account setup and authentication  
  - **Sticky Note3**  
    - Content: Instructions for enabling Vertex AI and configuring Google Cloud permissions

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                          | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                   |
|---------------------------|------------------------------------|----------------------------------------|-----------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| Get PDF or Images          | Google Drive Trigger                | Trigger on new file upload              | —                     | Route based on PDF or Image| Sticky Note1: Instructions for Google Drive folder setup and service account sharing                          |
| Route based on PDF or Image| Switch                             | Routes files by MIME type               | Get PDF or Images      | Download PDF, Download Image|                                                                                                              |
| Download PDF              | Google Drive                       | Downloads PDF files                     | Route based on PDF or Image | Extract data from PDF     |                                                                                                              |
| Extract data from PDF      | Extract From File                  | Extracts text from PDF                  | Download PDF           | Send data to A.I.          |                                                                                                              |
| Send data to A.I.          | HTTP Request                      | Sends extracted text to OpenRouter AI  | Extract data from PDF  | Convert to CSV             | Sticky Note2: Setup OpenRouter account; use Header Auth with Bearer token                                   |
| Convert to CSV             | Convert To File                   | Converts AI text response to CSV file  | Send data to A.I.      | Upload to Google Drive     |                                                                                                              |
| Upload to Google Drive     | Google Drive                      | Uploads CSV file to Drive output folder| Convert to CSV         | —                         |                                                                                                              |
| Download Image             | Google Drive                      | Downloads image files                   | Route based on PDF or Image | Vertex A.I. extract text |                                                                                                              |
| Vertex A.I. extract text   | Langchain chainLlm (Google Gemini)| Sends image to Vertex AI Gemini for extraction | Download Image    | Convert to CSV2            | Sticky Note3: Enable Vertex AI API and set IAM permissions on Google Cloud                                  |
| Convert to CSV2            | Convert To File                   | Converts Vertex AI response to CSV     | Vertex A.I. extract text | Upload to Google Drive1    |                                                                                                              |
| Upload to Google Drive1    | Google Drive                      | Uploads CSV to Drive output folder     | Convert to CSV2        | —                         |                                                                                                              |
| Sticky Note               | Sticky Note                      | Documentation and instructions         | —                     | —                         | Sticky Note: Overview and blog link                                                                          |
| Sticky Note1              | Sticky Note                      | Documentation                          | —                     | —                         | Sticky Note1: Google Drive folder and service account setup                                                  |
| Sticky Note2              | Sticky Note                      | Documentation                          | —                     | —                         | Sticky Note2: OpenRouter API setup instructions                                                             |
| Sticky Note3              | Sticky Note                      | Documentation                          | —                     | —                         | Sticky Note3: Vertex AI setup instructions                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure to watch a specific folder for new files (folder ID: `1HOeRP5iwccg93UPUYmWYD7DyDmRREkhj`)  
   - Poll every minute  
   - Authenticate via Google Service Account with Drive access to folder  

2. **Add a Switch Node to Route by MIME Type**  
   - Type: Switch  
   - Set rules:  
     - If MIME type equals `application/pdf`, output key: "pdf"  
     - If MIME type contains `image/`, output key: "image"  
   - Use expression to get MIME type from trigger node: `{{$json.mimeType}}`  

3. **Create Google Drive Download Nodes for PDFs and Images**  
   - Two separate Google Drive nodes:  
     - For PDFs: Operation "download", file ID from trigger node  
     - For Images: Operation "download", file ID from trigger node  
   - Use the same Google Service Account credentials  

4. **PDF Processing Branch**  
   - Add Extract From File node:  
     - Operation: pdf  
     - Input: binary data from PDF download node  
   - Add HTTP Request node to send extracted text to OpenRouter AI:  
     - Method: POST  
     - URL: `https://openrouter.ai/api/v1/chat/completions`  
     - JSON body with prompt instructing model to parse bank statement text and output CSV with category column  
     - Authentication: Header Auth with name "Authorization" and value "Bearer {your API token}"  
   - Add Convert To File node to convert AI response text to CSV binary  
   - Add Google Drive node to upload CSV:  
     - Upload to folder ID `1Zo4OFCv1qWRX1jo0VL_iqUBf4v0fZEXe` ("CSV Exports")  
     - Filename: dynamic date (e.g., `{{$today}}`)  
     - Use Google Service Account credentials  

5. **Image Processing Branch**  
   - Add Langchain chainLlm node configured for Google Gemini model:  
     - Model: `models/gemini-1.5-pro-latest` (ensure correct credentials)  
     - Prompt: instruct Gemini to extract transactions from image as CSV with category column  
     - Use HumanMessagePromptTemplate to send image binary data as input  
   - Add Convert To File node to convert Gemini response text to CSV binary  
   - Add Google Drive node to upload CSV (same folder and naming as PDF branch)  

6. **Credentials Setup**  
   - Google Service Account with Drive API enabled and permissions to both input and output folders  
   - OpenRouter API account with API token for HTTP header authentication  
   - Google Cloud project with Vertex AI enabled for the Gemini model  
     - Assign appropriate IAM roles to service account to access Vertex AI  

7. **Testing and Validation**  
   - Upload sample PDF and image files to the watched folder  
   - Confirm workflow triggers and correctly routes, processes, and uploads CSV files  
   - Check for error handling in case of API failures, permissions issues, or malformed inputs  

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow blog post explaining use case and detailed setup steps                                                                          | https://rumjahn.com/how-to-extract-pdf-and-image-text-into-csv-using-n8n-without-manual-data-entry/               |
| Google Drive folder setup requires sharing with the Google Cloud service account email (e.g., n8n-server@...)                             | Included in Sticky Note1                                                                                          |
| OpenRouter API requires Header Authentication with Bearer token ("Authorization" header)                                                 | Included in Sticky Note2                                                                                          |
| Enable Vertex AI API and grant Vertex AI access to service account in Google Cloud IAM                                                    | Included in Sticky Note3                                                                                          |
| For Vertex AI Gemini model usage, ensure Google Cloud project has required quotas and billing enabled                                    | Best practice for production workloads                                                                           |
| CSV files are named by current date; consider adding timestamp or unique suffix to avoid overwriting                                      | Workflow improvement suggestion                                                                                   |
| AI prompt templates can be adjusted to extract different data or modify CSV output format                                                | Flexible to user needs                                                                                            |

---

This structured overview and detailed breakdown provide comprehensive documentation to understand, modify, and reproduce this text extraction workflow using n8n integrating Google Drive, Vertex AI (Gemini), and OpenRouter AI.