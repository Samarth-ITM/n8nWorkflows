Generate video from prompt using Vertex AI Veo 3 and upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-video-from-prompt-using-vertex-ai-veo-3-and-upload-to-google-drive-5228


# Generate video from prompt using Vertex AI Veo 3 and upload to Google Drive

---

## 1. Workflow Overview

This workflow automates the process of generating a video from a text prompt using Google Cloud’s Vertex AI Veo 3 model and then uploading the generated video to Google Drive. It is designed to accept user input via a form (text prompt and GCP access token), invoke the Vertex AI long-running video generation model, wait for completion, convert the output from Base64 to a video file, and finally upload the video file to a specified Google Drive folder.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user inputs via an n8n form trigger node.
- **1.2 Configuration Setup:** Sets environment variables and parameters needed for Vertex AI API calls.
- **1.3 Video Generation Request:** Sends the text prompt to Vertex AI to start the asynchronous video generation.
- **1.4 Polling for Completion:** Waits and then fetches the video generation operation result.
- **1.5 File Conversion:** Converts the Base64-encoded video data into a binary video file.
- **1.6 Upload to Google Drive:** Uploads the video file to a specific Google Drive folder.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving user inputs—specifically, a text prompt and a Google Cloud Platform (GCP) access token—via a web form.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **Node Name:** On form submission  
- **Type:** Form Trigger  
- **Technical Role:** Listens for form submissions; acts as the workflow’s trigger.  
- **Configuration:**  
  - Webhook ID assigned for external form submissions.  
  - Form titled “Google Vertex AI” with two required fields: “Prompt” (text prompt for video generation) and “YOUR_ACCESS_TOKEN” (GCP OAuth2 access token).  
  - Form description: “Google Vertex”.  
- **Key Expressions/Variables:** None at input stage; data captured is passed downstream as JSON.  
- **Input Connections:** None (trigger node).  
- **Output Connections:** Connects to “Setting” node.  
- **Version Requirements:** n8n version supporting formTrigger v2.2 or higher.  
- **Edge Cases / Failures:**  
  - Missing or invalid prompt or access token input.  
  - Unauthorized access if webhook is improperly secured.  
- **Sub-workflow:** None.

---

### 2.2 Configuration Setup

**Overview:**  
This block sets essential environment variables and parameters for the Vertex AI API, including project ID, model version, location, API endpoint, and dynamically assigns the prompt and access token from the form input.

**Nodes Involved:**  
- Setting

**Node Details:**

- **Node Name:** Setting  
- **Type:** Set  
- **Technical Role:** Defines workflow variables and constants required for downstream API calls.  
- **Configuration:**  
  - Assigns the following variables:  
    - `PROJECT_ID`: User must enter their Google Cloud project ID.  
    - `MODEL_VERSION`: Set statically to “veo-3.0-generate-preview”.  
    - `LOCATION`: User’s Google Cloud location (region).  
    - `TEXT_PROMPT`: Dynamically pulled from the form input’s “Prompt” field.  
    - `IMAGE_COUNT`: Set to string “1” (number of videos/images to generate).  
    - `API_ENDPOINT`: Derived from the LOCATION as `<YOUR_LOCATION>-aiplatform.googleapis.com`.  
    - `ACCESS_TOKEN`: Dynamically pulled from the form input’s “YOUR_ACCESS_TOKEN” field.  
- **Key Expressions/Variables:**  
  - `={{ $json.Prompt }}` for prompt.  
  - `={{ $json.YOUR_ACCESS_TOKEN }}` for access token.  
- **Input Connections:** From “On form submission”.  
- **Output Connections:** To “Vertex AI-VEO3”.  
- **Version Requirements:** Supports n8n Set node v3.4 or higher.  
- **Edge Cases / Failures:**  
  - Missing or incorrect project ID or location leading to invalid API endpoint.  
  - Empty or malformed prompt or access token causing API authorization or request errors.  
- **Sub-workflow:** None.

---

### 2.3 Video Generation Request

**Overview:**  
This block sends a POST request to the Vertex AI `predictLongRunning` endpoint to start the asynchronous video generation process based on the provided text prompt. It initiates the long-running operation to generate a video.

**Nodes Involved:**  
- Vertex AI-VEO3

**Node Details:**

- **Node Name:** Vertex AI-VEO3  
- **Type:** HTTP Request  
- **Technical Role:** Makes authenticated HTTP POST request to start video generation using Vertex AI API.  
- **Configuration:**  
  - URL template dynamically composed using project ID, location, model version, and API endpoint from previous node.  
  - Method: POST  
  - Headers include `Content-Type: application/json` and `Authorization: Bearer <ACCESS_TOKEN>`.  
  - JSON body includes:  
    - Endpoint string (hardcoded project/model string).  
    - Instances array with the prompt.  
    - Parameters: aspect ratio (16:9), sample count (1), duration (8 seconds), person generation allowed, watermark enabled, reason included, audio generated.  
- **Key Expressions/Variables:**  
  - URL: `=https://{{ $json.API_ENDPOINT }}/v1/projects/{{ $json.PROJECT_ID }}/locations/{{ $json.LOCATION }}/publishers/google/models/{{ $json.MODEL_VERSION }}:predictLongRunning`  
  - Authorization header: `=Bearer {{ $json.ACCESS_TOKEN }}`  
  - Body prompt: `{{ $json.TEXT_PROMPT }}`  
- **Input Connections:** From “Setting”.  
- **Output Connections:** To “Wait” node.  
- **Version Requirements:** HTTP Request node v4.2 or higher recommended for header/body customization.  
- **Edge Cases / Failures:**  
  - Authorization failure due to expired or invalid access token.  
  - API quota or project permission issues.  
  - Invalid prompt formatting causing API rejections.  
  - Network timeouts.  
- **Sub-workflow:** None.

---

### 2.4 Polling for Completion

**Overview:**  
This block waits for a fixed 2-minute interval, then sends a request to check the status of the long-running video generation operation, retrieving the final result once ready.

**Nodes Involved:**  
- Wait  
- Vertex AI-fetch

**Node Details:**

- **Node Name:** Wait  
- **Type:** Wait  
- **Technical Role:** Pauses workflow execution for 2 minutes to allow video generation to process.  
- **Configuration:**  
  - Unit: minutes  
  - Amount: 2  
- **Input Connections:** From “Vertex AI-VEO3”.  
- **Output Connections:** To “Vertex AI-fetch”.  
- **Version Requirements:** Wait node v1.1 or higher.  
- **Edge Cases / Failures:**  
  - Insufficient wait time leading to premature fetch and incomplete results.  
  - Workflow timeout if wait too long or API delays.  

- **Node Name:** Vertex AI-fetch  
- **Type:** HTTP Request  
- **Technical Role:** Queries the Vertex AI API’s `fetchPredictOperation` endpoint to obtain the video generation operation status and result.  
- **Configuration:**  
  - URL dynamically constructed from Setting node parameters (project ID, location, model version, API endpoint).  
  - POST method with JSON body containing the `operationName` extracted from the previous node’s response.  
  - Headers include `Content-Type: application/json` and authorization using the access token from the form submission node.  
- **Key Expressions/Variables:**  
  - URL: `=https://{{ $('Setting').item.json.API_ENDPOINT }}/v1/projects/{{ $('Setting').item.json.PROJECT_ID }}/locations/{{ $('Setting').item.json.LOCATION }}/publishers/google/models/{{ $('Setting').item.json.MODEL_VERSION }}:fetchPredictOperation`  
  - Authorization header: `=Bearer {{ $('On form submission').item.json.YOUR_ACCESS_TOKEN }}`  
  - Body: `{"operationName": "{{ $json.name }}"}` where `$json.name` is the operation name from the previous response.  
- **Input Connections:** From “Wait”.  
- **Output Connections:** To “Convert to File”.  
- **Version Requirements:** HTTP Request node v4.2 or higher.  
- **Edge Cases / Failures:**  
  - Operation not complete yet leading to empty or incomplete response.  
  - Invalid or expired access tokens.  
  - API errors or malformed operation name.  
- **Sub-workflow:** None.

---

### 2.5 File Conversion

**Overview:**  
This block converts the Base64-encoded video data returned from the Vertex AI operation into a binary video file format suitable for uploading.

**Nodes Involved:**  
- Convert to File

**Node Details:**

- **Node Name:** Convert to File  
- **Type:** Convert To File  
- **Technical Role:** Converts Base64 string representing the video bytes into a binary file within n8n.  
- **Configuration:**  
  - Operation: toBinary (convert JSON Base64 field to binary file)  
  - Source property: `response.videos[0].bytesBase64Encoded` (path to Base64 video data in JSON)  
- **Input Connections:** From “Vertex AI-fetch”.  
- **Output Connections:** To “Google Drive”.  
- **Version Requirements:** Convert To File node v1.1 or higher.  
- **Edge Cases / Failures:**  
  - Missing or malformed Base64 data causing conversion failure.  
  - Large file size may cause memory issues.  
- **Sub-workflow:** None.

---

### 2.6 Upload to Google Drive

**Overview:**  
This final block uploads the generated video file to a designated Google Drive folder, naming the file based on the form submission timestamp.

**Nodes Involved:**  
- Google Drive

**Node Details:**

- **Node Name:** Google Drive  
- **Type:** Google Drive  
- **Technical Role:** Uploads the binary video file to Google Drive with OAuth2 authentication.  
- **Configuration:**  
  - File name: dynamically set to the submission timestamp from “On form submission” node, suffixed with `.mp4`.  
  - Drive ID: set to “My Drive”.  
  - Folder ID: set to a specific folder (`1tzpmwAWiUGolnKciZvcCghB5obhPoXzL`).  
  - Credentials: Uses preconfigured OAuth2 credentials named “Google Drive account - Peakwave”.  
- **Key Expressions/Variables:**  
  - File name: `={{ $('On form submission').item.json.submittedAt }}.mp4`  
- **Input Connections:** From “Convert to File”.  
- **Output Connections:** None (terminal node).  
- **Version Requirements:** Google Drive node v3 or higher for OAuth2 support.  
- **Edge Cases / Failures:**  
  - Authentication failures if OAuth token expired.  
  - Folder or permission issues preventing upload.  
  - File name conflicts or invalid characters.  
- **Sub-workflow:** None.

---

## 3. Summary Table

| Node Name           | Node Type          | Functional Role                       | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                                                     |
|---------------------|--------------------|------------------------------------|---------------------|--------------------|--------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger       | Captures user text prompt & token  | None                | Setting            | Accepts a text prompt and a GCP access token via form.                                                                          |
| Setting             | Set                | Sets environment & API parameters   | On form submission  | Vertex AI-VEO3     | Setting GCP - PROJECT_ID, MODEL_VERSION, LOCATION, IMAGE_COUNT, API_ENDPOINT                                                    |
| Vertex AI-VEO3       | HTTP Request       | Sends prompt to Vertex AI Veo3      | Setting             | Wait               | Veo3 - Sends prompt to predictLongRunning, waits for rendering, fetches result                                                 |
| Wait                 | Wait               | Pauses execution to allow rendering | Vertex AI-VEO3      | Vertex AI-fetch    |                                                                                                                                |
| Vertex AI-fetch      | HTTP Request       | Fetches video generation result     | Wait                | Convert to File    |                                                                                                                                |
| Convert to File      | Convert To File    | Converts Base64 video to binary file| Vertex AI-fetch     | Google Drive       | Convert to Video file - Base64 Input Field: response.videos[0].bytesBase64Encoded                                               |
| Google Drive         | Google Drive       | Uploads video file to Drive folder  | Convert to File     | None               | Upload Video to Google Drive                                                                                                   |
| Sticky Note          | Sticky Note        | Notes on GCP settings                | None                | None               | Setting GCP - PROJECT_ID, MODEL_VERSION, LOCATION, IMAGE_COUNT, API_ENDPOINT                                                    |
| Sticky Note1         | Sticky Note        | Notes explaining Veo3 block          | None                | None               | Veo3 - Sends the prompt to the Veo3 using Vertex AI’s predictLongRunning endpoint. Waits for the video rendering to complete.  |
| Sticky Note2         | Sticky Note        | Notes on Convert to File node       | None                | None               | Convert to Video file - Base64 Input Field: response.videos[0].bytesBase64Encoded                                               |
| Sticky Note3         | Sticky Note        | Notes on Google Drive upload         | None                | None               | Upload Video to Google Drive                                                                                                   |
| Sticky Note4         | Sticky Note        | Notes on form input                  | None                | None               | Accepts a text prompt and a GCP access token via form.                                                                          |
| Sticky Note5         | Sticky Note        | Workflow process diagram             | None                | None               | Workflow Process [Google Drive image link](https://drive.google.com/thumbnail?id=1L9KKkuS0hk5LW9hpGJ_FB9giKYFZpmy4&sz=w1000) |
| Sticky Note6         | Sticky Note        | Output sample image                  | None                | None               | Output [Google Drive image link](https://drive.google.com/thumbnail?id=1Biq2vhbzaFLya1ZsF8PhGL1RRta7XkMK&sz=w1000)             |
| Sticky Note7         | Sticky Note        | How to get GCP Access Token          | None                | None               | How to get GCP Access Token: Use `gcloud auth print-access-token` command in VM/Cloud Shell                                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “On form submission” Node:**  
   - Type: Form Trigger  
   - Configure form with title “Google Vertex AI”.  
   - Add two required fields: “Prompt” (text), “YOUR_ACCESS_TOKEN” (text).  
   - Save and note the webhook URL.

2. **Create “Setting” Node:**  
   - Type: Set  
   - Add the following variables with values or expressions:  
     - `PROJECT_ID`: string, set to your Google Cloud project ID (e.g., `my-gcp-project`).  
     - `MODEL_VERSION`: string, set to `veo-3.0-generate-preview`.  
     - `LOCATION`: string, e.g., `us-central1`.  
     - `TEXT_PROMPT`: expression `={{ $json.Prompt }}` (from form input).  
     - `IMAGE_COUNT`: string, set to `1`.  
     - `API_ENDPOINT`: string, set to `<YOUR_LOCATION>-aiplatform.googleapis.com` (replace `<YOUR_LOCATION>` with your region).  
     - `ACCESS_TOKEN`: expression `={{ $json.YOUR_ACCESS_TOKEN }}` (from form input).  
   - Connect “On form submission” output to “Setting” input.

3. **Create “Vertex AI-VEO3” Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: expression to construct URL:  
     `=https://{{ $json.API_ENDPOINT }}/v1/projects/{{ $json.PROJECT_ID }}/locations/{{ $json.LOCATION }}/publishers/google/models/{{ $json.MODEL_VERSION }}:predictLongRunning`  
   - Headers:  
     - `Content-Type: application/json`  
     - `Authorization: Bearer {{ $json.ACCESS_TOKEN }}`  
   - Body (JSON, raw):  
     ```json
     {
       "endpoint": "projects/n8n-project-440404/locations/us-central1/publishers/google/models/veo-3.0-generate-preview",
       "instances": [
         {
           "prompt": {{ $json.TEXT_PROMPT }}
         }
       ],
       "parameters": {
         "aspectRatio": "16:9",
         "sampleCount": 1,
         "durationSeconds": "8",
         "personGeneration": "allow_all",
         "addWatermark": true,
         "includeRaiReason": true,
         "generateAudio": true
       }
     }
     ```  
   - Connect “Setting” output to “Vertex AI-VEO3” input.

4. **Create “Wait” Node:**  
   - Type: Wait  
   - Set to wait 2 minutes.  
   - Connect “Vertex AI-VEO3” output to “Wait” input.

5. **Create “Vertex AI-fetch” Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: expression:  
     `=https://{{ $('Setting').item.json.API_ENDPOINT }}/v1/projects/{{ $('Setting').item.json.PROJECT_ID }}/locations/{{ $('Setting').item.json.LOCATION }}/publishers/google/models/{{ $('Setting').item.json.MODEL_VERSION }}:fetchPredictOperation`  
   - Headers:  
     - `Content-Type: application/json`  
     - `Authorization: Bearer {{ $('On form submission').item.json.YOUR_ACCESS_TOKEN }}`  
   - Body (JSON):  
     ```json
     {
       "operationName": "{{ $json.name }}"
     }
     ```  
   - Connect “Wait” output to “Vertex AI-fetch” input.

6. **Create “Convert to File” Node:**  
   - Type: Convert To File  
   - Operation: toBinary  
   - Source Property: `response.videos[0].bytesBase64Encoded`  
   - Connect “Vertex AI-fetch” output to “Convert to File” input.

7. **Create “Google Drive” Node:**  
   - Type: Google Drive  
   - Operation: Upload file  
   - File Name: expression: `={{ $('On form submission').item.json.submittedAt }}.mp4`  
   - Drive ID: Select “My Drive” or specific drive.  
   - Folder ID: Enter the target Google Drive folder ID (e.g., `1tzpmwAWiUGolnKciZvcCghB5obhPoXzL`).  
   - Credentials: Set up and select Google Drive OAuth2 credentials.  
   - Connect “Convert to File” output to “Google Drive” input.

8. **Test Workflow:**  
   - Submit the form with a valid prompt and a valid GCP access token (get token via `gcloud auth print-access-token`).  
   - Check logs and Google Drive folder for video upload.

---

## 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| How to get GCP Access Token: Use `gcloud auth print-access-token` in VM or Cloud Shell.          | See Sticky Note7                                                                                     |
| Workflow Process Diagram available as image in Google Drive.                                     | https://drive.google.com/thumbnail?id=1L9KKkuS0hk5LW9hpGJ_FB9giKYFZpmy4&sz=w1000                    |
| Output preview image showing generated video file metadata/thumbnail.                           | https://drive.google.com/thumbnail?id=1Biq2vhbzaFLya1ZsF8PhGL1RRta7XkMK&sz=w1000                    |
| Vertex AI Veo 3 Model documentation - use predictLongRunning endpoint for video generation.      | [Google Cloud Vertex AI docs](https://cloud.google.com/vertex-ai/docs) (not linked in workflow)      |
| Ensure Google Drive OAuth2 credentials have write access to the target folder for uploads.       | Credential setup note                                                                                 |

---

**Disclaimer:** The content provided is exclusively generated from an n8n workflow and strictly adheres to content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.

---