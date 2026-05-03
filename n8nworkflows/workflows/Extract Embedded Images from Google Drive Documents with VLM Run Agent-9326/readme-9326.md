Extract Embedded Images from Google Drive Documents with VLM Run Agent

https://n8nworkflows.xyz/workflows/extract-embedded-images-from-google-drive-documents-with-vlm-run-agent-9326


# Extract Embedded Images from Google Drive Documents with VLM Run Agent

### 1. Workflow Overview

This n8n workflow automates the extraction of embedded images from Google Drive documents using a VLM Run AI agent, then downloads and saves those images back to a specified Google Drive folder. It is tailored for use cases such as processing receipts, reports, or other documents containing images that need to be extracted for further use or machine learning datasets.

The workflow is logically divided into the following blocks:

- **1.1 Monitor Uploads and File Download**: Watches a specific Google Drive folder for new files (e.g., receipts), then downloads each detected file for processing.
- **1.2 Image Extraction via VLM Run Agent**: Sends the downloaded file to a VLM Run AI agent configured to extract embedded image URLs from the document.
- **1.3 Receive Extracted Image URLs**: Accepts a callback POST request from VLM Run containing the extracted image URLs.
- **1.4 Split, Download, and Save Images**: Splits the list of image URLs, downloads each image file, and saves them individually to a target Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Monitor Uploads and File Download

**Overview:**  
This block continuously monitors a designated Google Drive folder for newly created files (e.g., newly uploaded receipts). Upon detection, it downloads the file binary content for subsequent AI processing.

**Nodes Involved:**  
- Monitor Uploads  
- Download File

**Node Details:**  

- **Monitor Uploads**  
  - *Type:* Google Drive Trigger  
  - *Role:* Triggers the workflow when a new file is created in the specified folder.  
  - *Configuration:*  
    - Event: `fileCreated`  
    - Poll interval: every minute  
    - Folder watched: Specific folder ID (`1S6baavqJn98MjUlbB6KtmARCWuWEekIZ`) named "test_data"  
  - *Credentials:* Google Drive OAuth2 account  
  - *Input/Output:* No input; outputs metadata including file ID for newly created file.  
  - *Failure cases:* Permission errors accessing folder, API rate limits, folder ID mismatch.  
  - *Notes:* Automatically triggers processing upon new file upload.

- **Download File**  
  - *Type:* Google Drive  
  - *Role:* Downloads the newly detected file as binary data for AI processing.  
  - *Configuration:*  
    - Operation: `download`  
    - File ID: dynamically taken from trigger node output (`{{$json.id}}`)  
    - Binary property name: `data`  
  - *Credentials:* Google Drive OAuth2 account (same as trigger)  
  - *Input:* File ID from “Monitor Uploads”  
  - *Output:* Binary file data under property `data`  
  - *Failure cases:* File not found, permission denied, download timeout.

---

#### 2.2 Image Extraction via VLM Run Agent

**Overview:**  
This block sends the downloaded file to a VLM Run AI agent configured to extract embedded images. The agent processes the document and asynchronously returns extracted image URLs via a webhook callback.

**Nodes Involved:**  
- Extract Images  
- Receive Image Links

**Node Details:**  

- **Extract Images**  
  - *Type:* VLM Run Agent node  
  - *Role:* Executes an AI agent prompt to extract image URLs from the document.  
  - *Configuration:*  
    - Operation: `executeAgent`  
    - Agent prompt: instructs the agent to extract images so URLs appear under `json.body.response.extracted_images`  
    - Agent callback URL: points to the n8n webhook node endpoint `/image-extract-via-agent`  
  - *Credentials:* VLM Run API key  
  - *Input:* File binary data from “Download File” node  
  - *Output:* Initiates the extraction job; does not provide immediate extracted URLs  
  - *Failure cases:* Agent API errors, invalid prompt, callback URL unreachable, authentication issues.

- **Receive Image Links**  
  - *Type:* Webhook (HTTP POST listener)  
  - *Role:* Receives asynchronous POST requests from VLM Run agent containing JSON with extracted image URLs.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: `/image-extract-via-agent`  
  - *Input:* JSON payload from VLM Run agent with field `body.response.extracted_images` (an array of URLs)  
  - *Output:* Passes extracted image URLs downstream  
  - *Failure cases:* Webhook not reachable externally, invalid JSON payload, missing expected fields.

---

#### 2.3 Split, Download, and Save Images

**Overview:**  
Once image URLs are received, this block splits the array of URLs for individual processing, downloads each image file, and saves it to a pre-defined Google Drive folder.

**Nodes Involved:**  
- Split Out  
- Download Image  
- Save Image

**Node Details:**  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Iterates over each image URL in the array `body.response.extracted_images` for individual processing.  
  - *Configuration:*  
    - Field to split out: `body.response.extracted_images`  
  - *Input:* JSON containing array of image URLs  
  - *Output:* Emits each URL as a separate item downstream  
  - *Failure cases:* Field missing or empty, non-array data.

- **Download Image**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the image file from the given URL.  
  - *Configuration:*  
    - URL: dynamically set to the current item URL (`{{$json['body.response.extracted_images']}}`)  
    - Response format: file (binary)  
  - *Input:* Single image URL from “Split Out” node  
  - *Output:* Image file binary under response data  
  - *Failure cases:* HTTP errors (404, 403), timeout, invalid URL.

- **Save Image**  
  - *Type:* Google Drive  
  - *Role:* Uploads the downloaded image binary to a specified Google Drive folder.  
  - *Configuration:*  
    - Drive: “My Drive”  
    - Folder ID: `1XD17_AMlTFIWCyhQjvbQnuGd7jkn91A-` (named “Extracted Image”)  
  - *Credentials:* Google Drive OAuth2 account  
  - *Input:* Image binary data from “Download Image”  
  - *Output:* Confirmation metadata of file saved  
  - *Failure cases:* Permission denied, incorrect folder ID, upload failures.

---

### 3. Summary Table

| Node Name        | Node Type               | Functional Role                             | Input Node(s)      | Output Node(s)    | Sticky Note                                                                                                                |
|------------------|-------------------------|---------------------------------------------|--------------------|-------------------|----------------------------------------------------------------------------------------------------------------------------|
| Monitor Uploads  | Google Drive Trigger     | Watches folder for new files                  | -                  | Download File     | * ⏱️ Checks folder every minute for new files; triggers processing automatically.                                           |
| Download File    | Google Drive             | Downloads new files as binary                  | Monitor Uploads     | Extract Images    | * ⬇️ Downloads files for AI processing.                                                                                     |
| Extract Images   | VLM Run Agent            | Sends file to AI agent to extract image URLs | Download File      | (Async) Receive Image Links | * VLM Run extracts images and posts URLs asynchronously to webhook.                                                        |
| Receive Image Links | Webhook                 | Receives extracted image URLs from VLM Run   | (Async from extract) | Split Out        | * Receives image URLs posted by VLM Run agent callback.                                                                     |
| Split Out       | Split Out                | Iterates over extracted image URLs            | Receive Image Links | Download Image    | * ✂️ Splits array of image URLs for individual download.                                                                    |
| Download Image  | HTTP Request             | Downloads individual images                     | Split Out          | Save Image        | * ⬇️ Downloads each image as a file.                                                                                        |
| Save Image      | Google Drive             | Saves downloaded images to target folder       | Download Image     | -                 | * 💾 Saves images to **Extracted Image** folder.                                                                             |
| Sticky Note     | Sticky Note              | Documentation notes                            | -                  | -                 | See details in node notes below.                                                                                            |
| Sticky Note1    | Sticky Note              | Documentation notes                            | -                  | -                 | See details in node notes below.                                                                                            |
| Sticky Note2    | Sticky Note              | Documentation notes                            | -                  | -                 | See details in node notes below.                                                                                            |
| Sticky Note4    | Sticky Note              | Documentation notes                            | -                  | -                 | See details in node notes below.                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node (Monitor Uploads)**
   - Type: Google Drive Trigger  
   - Set event to `fileCreated`  
   - Set polling interval to every minute  
   - Configure folder to watch by selecting the target folder ID (`1S6baavqJn98MjUlbB6KtmARCWuWEekIZ`)  
   - Attach Google Drive OAuth2 credentials with access to that folder  

2. **Create Google Drive Node (Download File)**
   - Type: Google Drive  
   - Set operation to `download`  
   - File ID parameter: set to expression `{{$json.id}}` from trigger output  
   - Set binary property name to `data`  
   - Use same Google Drive OAuth2 credentials as trigger  
   - Connect output of “Monitor Uploads” to input of “Download File”  

3. **Create VLM Run Node (Extract Images)**
   - Type: VLM Run Agent node  
   - Set operation to `executeAgent`  
   - Configure agent prompt to:  
     > "You are expert in extracting images from a document, complete the task accordingly so that urls can be extracted using json.body.response.extracted_images"  
   - Set agent callback URL to the webhook’s external URL that you will create in step 4 (e.g., `https://your-n8n-domain/webhook/image-extract-via-agent`)  
   - Set credentials for VLM Run API access  
   - Connect output of “Download File” to input of “Extract Images”  

4. **Create Webhook Node (Receive Image Links)**
   - Type: Webhook  
   - Set HTTP method to POST  
   - Set path to `image-extract-via-agent`  
   - No authentication unless desired  
   - This node will receive JSON from VLM Run agent with extracted image URLs in the field `body.response.extracted_images`  
   - Connect output to “Split Out” node (created next)  

5. **Create Split Out Node (Split Out)**
   - Type: Split Out  
   - Set field to split out to `body.response.extracted_images` (path to array of URLs)  
   - Connect output of “Receive Image Links” to “Split Out”  

6. **Create HTTP Request Node (Download Image)**
   - Type: HTTP Request  
   - Set URL parameter to expression `{{$json['body.response.extracted_images']}}` (current split item)  
   - Set response format to `file` (to download binary)  
   - Connect “Split Out” output to “Download Image” input  

7. **Create Google Drive Node (Save Image)**
   - Type: Google Drive  
   - Set operation to `upload` (default)  
   - Set destination drive to “My Drive”  
   - Set folder ID to `1XD17_AMlTFIWCyhQjvbQnuGd7jkn91A-` (folder named “Extracted Image”)  
   - Use same Google Drive OAuth2 credentials as previous nodes  
   - Connect “Download Image” output to “Save Image” input  

8. **Configure Credentials**
   - Google Drive OAuth2 with permissions for both watched folder and upload folder  
   - VLM Run API key with permissions to execute agents and receive callbacks  

9. **Test the Workflow**
   - Upload a test document with embedded images to the watched Google Drive folder  
   - Confirm the VLM Run agent is triggered and sends extracted image links to webhook  
   - Verify images are downloaded and saved to the target Google Drive folder  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| **Image Extraction Pipeline** Flow: Google Drive upload → VLM Run extracts images → posts to n8n webhook → images split, downloaded, and saved to target Drive folder. Use cases include receipts from PDFs, report images, and ML-ready assets. Requires VLM Run API credentials, Google Drive OAuth2, reachable webhook URL, and valid folder IDs. | Sticky Note on workflow overview node |
| **Extract Images via VLM Run – Setup Guide:** 1) Set up n8n webhook and copy URL. 2) Configure VLM Run agent with image extraction prompt. 3) Paste n8n webhook URL as callback. 4) Run job on document. 5) Receive extracted URLs in n8n. | Sticky Note near VLM Run node |
| **Split, Download, Save:** Split Out node iterates each link from extracted images array; HTTP Request downloads each image; Google Drive node saves each image to “Extracted Image” folder. Example JSON provided for links array format. | Sticky Note near Split Out node |
| **Monitor & Download:** Google Drive Trigger polls every minute in specified folder for new files; downloads files for AI processing. | Sticky Note near Google Drive Trigger node |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.